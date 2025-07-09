# CTXDOATD Reimplementation: Algorithmic Analysis & Synthesis

This document details the synthesized algorithms for a potential reimplementation of an interference analysis tool similar to CTXDOATD, based on "Book 1.txt" (CTX Curve Dataset Generation Guidelines) and "Book 2.txt" (MICS Reference Manual excerpts, including Annexes 2A-I and 2A-II).

## Overall Reimplementation Strategy

The system aims to calculate a CTX curve (permissible interference vs. frequency separation) from Victim and Interferer datasets. For dual conversion receivers, this involves:
1.  Parsing input datasets.
2.  Calculating a "main co-channel interference objective curve" for a specific assumed Intermediate Frequency (`XIF_current`).
3.  Adjusting this main curve for IF and Image spurious responses based on `XIF_current`.
4.  Orchestrating multiple runs (steps 2-3) for different potential spurious `XIF_current` values relevant to dual conversion.
5.  Generating a final composite CTX curve from the worst-case objectives of all runs.

## I. Core Calculation of "Main Co-channel Objective"

### A. For Digital Victims (D/D or A/D Interference)
*Source: Book 1, Sec 3.2.3 & 2.9; Book 2, Annex 2A-II*

**Principle:** Permissible interference at the demodulator slicer (`P_interf_slicer_target`) is set by `THDCRIT` relative to the thermal noise floor. The permissible input interference `ILVL(fs)` is this target power adjusted by how much normalized interferer power passes through victim filters at separation `fs`.

**Algorithm:**

1.  **Victim Characterization:**
    *   Inputs: Selectivity `H_v(f)` (power transfer function), Noise Figure `NF_v` (dB), `THDCRIT_v` (dB).
    *   Equivalent Noise Bandwidth `B_v` (Hz):
        *   `B_3dB` = 3dB bandwidth of `H_v(f)`.
        *   `B_brickwall = integral |H_v(f)|^2 df` (Equivalent Noise Bandwidth).
        *   `B_v = min(B_3dB, B_brickwall)`. (Based on Book 1, Sec 2.9 inference).
    *   Thermal Noise Power at Slicer `P_thermal_v_slicer` (mW):
        `P_thermal_v_slicer = (1.38054e-23 J/K) * (290 K) * B_v * (10^(NF_v / 10)) * 1000`.

2.  **Target Interference at Slicer:**
    *   Degradation factor `a = 10^(THDCRIT_v / 10) - 1`. (e.g., for 1 dB, `a approx 0.259`).
    *   `P_interf_slicer_target = a * P_thermal_v_slicer` (mW).

3.  **Interferer Characterization:**
    *   Input: Interferer PSD `S_i(f)` (e.g., W/Hz).
        *   D/D: Empirical dataset, scaled by total received power.
        *   A/D: Calculated analog FM spectrum (see Section I.B.2).
    *   Normalized PSD shape: `s_i_norm(f) = S_i(f) / (integral S_i(f) df)`.

4.  **Calculate Permissible Input Interference `ILVL(fs)` (dBm):**
    *   For each frequency separation `fs`:
        *   `Filtered_Norm_Interf_Power_at_Slicer(fs) = numerical_integral(s_i_norm(f) * |H_v(f-fs)|^2 df)`.
            *This requires numerical integration (e.g., trapezoidal rule) after aligning/interpolating `s_i_norm` and shifted `H_v` data points.*
        *   `Permissible_Input_Interf_Power_Linear(fs) [mW] = P_interf_slicer_target [mW] / Filtered_Norm_Interf_Power_at_Slicer(fs)`.
        *   `ILVL(fs) [dBm] = 10 * log10(Permissible_Input_Interf_Power_Linear(fs))`.
    *   Output: `ILVL(fs)` array (the main co-channel CTX objective).

### B. For Analog FDM-FM Victims (A/A or D/A Interference)
*Source: Book 2, Annex 2A-I; Book 1, FM equations page*

**Principle:** Calculate interference noise in a victim's baseband voice channel. Determine input I or C/I meeting a noise objective (e.g., 4 dBrnCO) or threshold degradation.

**Algorithm Outline:**

1.  **Victim & Interferer Analog Characterization:**
    *   Inputs: `N` (channels), `fmin`, `fm`, `Delta_sigma_rms_per_channel`, `NF` (victim), `XIF` (victim), pre-emphasis `Hp(f)`. Victim filtering (specific or default per Annex 2A-I, Sec 7).

2.  **Generate Power Spectral Densities `P1(f)` (desired), `P2(f)` (interferer):**
    *   **Analog Signals:** (Victim `P1(f)`; Interferer `P2(f)` if A/A)
        *   Calculate modulation index `m`.
        *   Low `m` (<=0.2): Medhurst-derived formulas (Annex 2A-I, Sec 3.1; Book 1 FM page for `R_phi_sq(o)`).
        *   Large `m` (>=1.5): Gaussian approximation (Annex 2A-I, Sec 3.2).
        *   Intermediate `m`: Interpolate from Hamer's tables (Annex 2A-I, Sec 3.3).
    *   **Digital Interferer (D/A):** Use empirical PSD dataset for `P2(f)`.

3.  **Calculate Convolved Baseband Interference Spectrum `I_bb(f_ch, fs)`:**
    *   For each victim baseband channel `f_ch` and carrier separation `fs`.
    *   Apply victim receiver filtering to `P2(f-fs)` if `fs` is large.
    *   `I_bb(f_ch, fs) = numerical_integral(P1(f') * P2_filtered(f' - (f_ch - fs)) df')`. (This is the conceptual form from Pontano et al. via Annex 2A-I, Sec 4. Precise implementation including `I_spike` for beat components is complex and likely needs details from referenced papers.)

4.  **Calculate NPR0(fs, f_ch) (Noise Power Ratio for C/I=0dB):**
    *   Using formula from Annex 2A-I, Sec 5, incorporating `I_bb(f_ch, fs)`.

5.  **Determine Objective (C/I or I) for `fs`:**
    *   **Long Haul (e.g., 4 dBrnCO -> S/N_target = 84 dB):**
        `C/I_req(fs, f_ch) = S/N_target - (NPR0 [dB] + BWR [dB] - NLR [dB])`.
        (`BWR` and `NLR` are standard terms defined in Annex 2A-I).
    *   **Short Haul (e.g., 1 dB noise degradation):**
        `I_obj(fs, f_ch) [dBm] = NPR0 [dB] - 139.1 + NF1 [dB] - 20log(Delta_sigma1/fm1) + BWR [dB] - NLR [dB] - 20logHp1(fm1) - 5.87 [dB]`.
    *   Select the worst objective across relevant `f_ch` for the current `fs`.
    *   Output: `C/I_req(fs)` or `ILVL(fs)` array.

## II. Spurious Response Adjustment Module

Applied to the `Main_Objective_Curve(fs)` from Part I.
*Source: Book 1, Sec 4.0, 5.0*

**Inputs:**
*   `Main_Objective_Curve(fs)`
*   `XIF_current` (the IF value used for this conceptual run)
*   `AI70_eff` (Effective IF response factor for `XIF_current`)
*   `AI140_eff` (Effective Image response factor for `XIF_current`)

**Algorithm:**

1.  Initialize `Adjusted_Curve(fs) = Main_Objective_Curve(fs)`.
2.  **IF Response Modification:**
    *   Create `IF_Contribution_Curve(f') = Main_Objective_Curve(f') - AI70_eff`. (where `f'` is separation from `XIF_current`).
    *   For `fs` in range `XIF_current ± XIF_current/2`:
        `Adjusted_Curve(fs) = min(Adjusted_Curve(fs), IF_Contribution_Curve_aligned_to_fs)`.
3.  **Image Response Modification:**
    *   Create `Image_Contribution_Curve(f'') = Main_Objective_Curve(f'') - AI140_eff`. (where `f''` is separation from `2*XIF_current`).
    *   For `fs` in range `2*XIF_current ± XIF_current/2`:
        `Adjusted_Curve(fs) = min(Adjusted_Curve(fs), Image_Contribution_Curve_aligned_to_fs)`.
4.  Output: `Final_Run_Curve(fs) = Adjusted_Curve(fs)`.

## III. Dual Conversion Orchestration & Composite Curve Generation
*Source: Book 1, Sec 6.0 and overall methodology*

**Algorithm:**

1.  **Identify Spurious Frequencies:**
    *   From receiver parameters (Fc, IF1, LO1, IF2, LO2), generate `XIF_spurious_list` using formulas from Book 1, Tables 1 & 3 (derived from Appendix I). Include IF1 and IF2.
2.  **Iterate for Each Spurious Frequency:**
    *   Initialize an empty list `All_Run_Output_Curves`.
    *   For each `XIF_k` in `XIF_spurious_list`:
        *   Prepare/select `AI70_k_eff` and `AI140_k_eff` corresponding to `XIF_k`. (This might involve looking up base victim RF filter attenuation at `XIF_k` and `2*XIF_k` and applying the +20dB rule for IF, or using pre-calculated Specific Factors if available. If using the "simplified practical approach" from Book 1 Sec 6.4, this step involves choosing the more conservative of IF1/IF2 derived factors).
        *   Calculate `Main_Objective_Curve_k` using algorithm I.A (Digital Victim) or I.B (Analog Victim), with `XIF_current = XIF_k`.
        *   Calculate `RunOutputCurve_k` by applying Spurious Adjustments (algorithm II) using `Main_Objective_Curve_k`, `XIF_k`, `AI70_k_eff`, `AI140_k_eff`.
        *   Add `RunOutputCurve_k` to `All_Run_Output_Curves`.
3.  **Generate Final Composite CTX Curve:**
    *   For each frequency point `fs`:
        `Composite_CTX(fs) = min(curve(fs) for curve in All_Run_Output_Curves)`.
    *   Output: `Composite_CTX(fs)`.

## Feasibility and Remaining Challenges

*   **Digital Victim Path:** Reasonably well-defined. Key challenges:
    *   Robust numerical integration for convolution.
    *   Precise method for calculating `B_equiv` (Equivalent Noise Bandwidth).
*   **Analog Victim Path:** Significantly more complex.
    *   Requires detailed implementation of FM spectral analysis (Medhurst, Hamer, Gaussian approx.) and baseband interference convolution (Pontano). This likely needs access to the referenced academic papers for full algorithm details.
    *   Precise handling of `I_spike` (carrier beat component).
*   **Spurious/Dual Conversion Framework:** The logic is clear once the main objective calculation is solved.
*   **Data Handling:**
    *   Parsing input datasets as per Book 1 format.
    *   Implementing data interpolation for mismatched frequency points in PSD/selectivity arrays.
    *   Implementing data extrapolation/truncation rules.
*   **Validation Data:** Essential. Example curves in Book 1 (Figures 6-11 for "Equipment #1") are a starting point but require precise reconstruction of "Equipment #1" input data.
*   **Missing Appendix I (Book 1):** This appendix, detailing the case studies for spurious frequency derivations, would be very helpful for confirming the conditions for each formula variant.

This synthesized algorithm provides a roadmap for reimplementation. The digital analysis path is more directly derivable from the provided documents than the analog path.
