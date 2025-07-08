# Proposed Relational Database Structure for CTX Analysis Data

Based on the analysis of "Book 1.txt" (CTX Curve Dataset Generation Guidelines) and "Book 2.txt" (MICS Reference Manual excerpts), the following relational database structure is proposed to store data required for interference calculations using CTXDOATD and the output of such calculations.

## I. Core Equipment & System Tables

These tables store fundamental characteristics of radio equipment, traffic types, and antennas.

### `EquipmentMasters`
Corresponds to `sd_eqpt` in MICS. Stores master records for equipment.

-   `EquipmentID` (Primary Key, e.g., INT AUTO_INCREMENT)
-   `EquipmentCode` (VARCHAR, UNIQUE, e.g., "RD-U6C" - the unique code for the equipment model)
-   `ManufacturerCode` (VARCHAR, Foreign Key to `Manufacturers.ManufacturerCode`)
-   `ModelName` (VARCHAR)
-   `EquipmentType` (VARCHAR, e.g., "Digital Radio", "Analog Radio")
-   `DefaultTrafficCode` (VARCHAR, Foreign Key to `TrafficTypes.TrafficCode`)
-   `FrequencyStability_Percent` (DECIMAL)
-   `DefaultIF1_MHz` (DECIMAL, First Intermediate Frequency)
-   `DefaultIF2_MHz` (DECIMAL, Second Intermediate Frequency, nullable for single conversion)
-   `DefaultReceiverNoiseFigure_dB` (DECIMAL)
-   `DefaultAuthorizedBandwidth_MHz` (DECIMAL)
-   `Notes` (TEXT)
-   `CTX_CrossReferenceEquipmentCode` (VARCHAR, Foreign Key to `EquipmentMasters.EquipmentCode`, nullable)

### `TrafficTypes`
Corresponds to `sd_traf` in MICS. Defines different types of traffic.

-   `TrafficCode` (Primary Key, VARCHAR, e.g., "D7138", "A1200")
-   `Description` (VARCHAR, e.g., "3DS3", "1200 VF Channels")
-   `ModulationType` (VARCHAR, e.g., "QAM", "PSK", "FM")
-   `ChannelCapacity` (INT, e.g., number of voice channels)
-   `BasebandMin_MHz` (DECIMAL, nullable, for analog)
-   `BasebandMax_MHz` (DECIMAL, nullable, for analog)
-   `RMSDeviation_kHz` (DECIMAL, nullable, for analog)
-   `CTX_CrossReferenceTrafficCode` (VARCHAR, Foreign Key to `TrafficTypes.TrafficCode`, nullable)

### `Antennas`
Corresponds to `sd_ante` in MICS. Stores antenna master records.

-   `AntennaCode` (Primary Key, VARCHAR)
-   `ManufacturerCode` (VARCHAR, Foreign Key to `Manufacturers.ManufacturerCode`)
-   `ModelName` (VARCHAR)
-   `Gain_dBi` (DECIMAL)
-   `Beamwidth_deg` (DECIMAL)
-   `FrontToBackRatio_dB` (DECIMAL)
-   `PolarizationType` (VARCHAR, e.g., "H", "V", "Dual")
-   `FrequencyBandCode` (VARCHAR, Foreign Key to `FrequencyBands.FrequencyBandCode`)

### `AntennaPatterns`
Corresponds to `sd_antd` in MICS. Stores antenna discrimination patterns.

-   `AntennaPatternID` (Primary Key, e.g., INT AUTO_INCREMENT)
-   `AntennaCode` (VARCHAR, Foreign Key to `Antennas.AntennaCode`)
-   `Angle_deg` (DECIMAL, Off-axis angle)
-   `DiscriminationCoPolH_dB` (DECIMAL)
-   `DiscriminationCrossPolH_dB` (DECIMAL)
-   `DiscriminationCoPolV_dB` (DECIMAL)
-   `DiscriminationCrossPolV_dB` (DECIMAL)

### `Manufacturers`
(Implied necessity for codes)
-   `ManufacturerCode` (Primary Key, VARCHAR)
-   `ManufacturerName` (VARCHAR)

### `FrequencyBands`
(Implied necessity for codes, corresponds to `sd_band`)
-   `FrequencyBandCode` (Primary Key, VARCHAR)
-   `BandName` (VARCHAR, e.g., "6 GHz Lower")
-   `FreqMin_MHz` (DECIMAL)
-   `FreqMax_MHz` (DECIMAL)
-   `AdjacentBandCodes` (VARCHAR, list of codes)


## II. Victim/Interferer Dataset Parameters (for CTXDOATD input)

These tables store the specific parameters used for a particular CTXDOATD analysis run.

### `VictimDatasetRuns`
Stores the header/summary information for a victim dataset used in a specific run.

-   `VictimRunID` (Primary Key, e.g., INT AUTO_INCREMENT)
-   `VictimEquipmentCode` (VARCHAR, Foreign Key to `EquipmentMasters.EquipmentCode`)
-   `VictimNameInRun` (VARCHAR, e.g., "Equipment #1" as used in Book 1 examples)
-   `RunDate` (DATE)
-   `AuthorizedBandwidth_MHz` (DECIMAL, ABW - specific for this run)
-   `NoiseFigure_dB` (DECIMAL, NF - specific for this run)
-   `IntermediateFrequency_MHz` (DECIMAL, XIF - the IF used for this specific run)
-   `AI70_dB` (DECIMAL, RF filter response at XIF, or SpecificIF-20dB)
-   `AI140_dB` (DECIMAL, RF filter response at 2*XIF, or SpecificImage)
-   `ThresholdDegradationCriterion_dB` (DECIMAL, THDCRIT)
-   `Comments` (TEXT)

### `VictimSelectivityData`
Stores the frequency vs. frequency response data points for a victim dataset run.

-   `SelectivityDataID` (Primary Key, e.g., INT AUTO_INCREMENT)
-   `VictimRunID` (INT, Foreign Key to `VictimDatasetRuns.VictimRunID`)
-   `FrequencyOffset_MHz` (DECIMAL)
-   `Attenuation_dB` (DECIMAL)
-   `PointOrder` (INT, to maintain sequence of data points)

### `InterfererDatasetRuns`
Stores the header/summary information for an interferer dataset used in a specific run.

-   `InterfererRunID` (Primary Key, e.g., INT AUTO_INCREMENT)
-   `InterfererEquipmentCode` (VARCHAR, Foreign Key to `EquipmentMasters.EquipmentCode`)
-   `InterfererNameInRun` (VARCHAR)
-   `RunDate` (DATE)
-   `Comments` (TEXT)

### `InterfererPSD_Data`
Stores the Power Spectral Density (PSD) data points for an interferer dataset run.

-   `PSD_DataID` (Primary Key, e.g., INT AUTO_INCREMENT)
-   `InterfererRunID` (INT, Foreign Key to `InterfererDatasetRuns.InterfererRunID`)
-   `FrequencyOffset_MHz` (DECIMAL)
-   `PSD_Value_dB_per_ref_BW` (DECIMAL, e.g., dBm/4kHz, ensure reference BW is consistent or stored)
-   `PointOrder` (INT, to maintain sequence)

## III. CTX Curve Definitions and Output Data

These tables store predefined CTX curves and the results of CTXDOATD calculations.

### `CTX_Definitions`
Corresponds to `sd_ctx` in MICS. Header information for a CTX curve.

-   `CTX_ID` (Primary Key, e.g., INT AUTO_INCREMENT or a composite key of the next three)
-   `VictimTrafficCode` (VARCHAR, Foreign Key to `TrafficTypes.TrafficCode`)
-   `InterfererTrafficCode` (VARCHAR, Foreign Key to `TrafficTypes.TrafficCode`)
-   `VictimReceiverEquipmentCode` (VARCHAR, Foreign Key to `EquipmentMasters.EquipmentCode`)
-   `CoChannelObjective` (DECIMAL, C/I or I level at 0 Hz separation)
-   `ObjectiveType` (VARCHAR, e.g., 'C/I_dB', 'I_dBm')
-   `Description` (TEXT)
-   `WorstObjectiveValue` (DECIMAL, overall worst point on the curve)
-   `BestObjectiveValue` (DECIMAL, overall best point on the curve)
-   `IsDefaultCurve` (BOOLEAN, indicates if this is a default or cross-referenced CTX)

### `CTX_DataPoints`
Corresponds to `sd_ctxd` in MICS. Actual data points for a CTX curve.

-   `CTX_DataPointID` (Primary Key, e.g., INT AUTO_INCREMENT)
-   `CTX_ID` (INT, Foreign Key to `CTX_Definitions.CTX_ID`)
-   `FrequencySeparation_MHz` (DECIMAL)
-   `ObjectiveValue` (DECIMAL, value corresponding to `ObjectiveType` in `CTX_Definitions`)

### `CTX_RunOutputs`
Stores the raw output from each individual CTXDOATD run, especially important for dual conversion analysis.

-   `CTX_RunOutputID` (Primary Key, e.g., INT AUTO_INCREMENT)
-   `VictimRunID` (INT, Foreign Key to `VictimDatasetRuns.VictimRunID`)
-   `InterfererRunID` (INT, Foreign Key to `InterfererDatasetRuns.InterfererRunID`)
-   `RunDateTime` (DATETIME)
-   `IF_Used_For_Run_MHz` (DECIMAL, Critical: which IF was used as XIF for this specific run)
-   `OutputNotes` (TEXT, e.g., path to raw output file if not storing points directly)

### `CTX_RunOutputDataPoints`
(Alternative to just storing filename in `CTX_RunOutputs`, stores actual points from a single run)
-   `RunOutputDataPointID` (Primary Key, e.g., INT AUTO_INCREMENT)
-   `CTX_RunOutputID` (INT, Foreign Key to `CTX_RunOutputs.CTX_RunOutputID`)
-   `FrequencySeparation_MHz` (DECIMAL)
-   `CalculatedObjectiveValue` (DECIMAL)


### `CTX_CompositeCurves`
Stores metadata for the final composite curves derived for dual conversion receivers.

-   `CompositeCTX_ID` (Primary Key, e.g., INT AUTO_INCREMENT)
-   `VictimEquipmentCode_Actual` (VARCHAR, FK to `EquipmentMasters.EquipmentCode` - the actual dual-conversion equipment)
-   `InterfererEquipmentCode_Actual` (VARCHAR, FK to `EquipmentMasters.EquipmentCode`)
-   `GenerationDate` (DATE)
-   `Description` (TEXT, e.g., "Composite CTX for Dual Conversion Receiver XYZ vs. Interferer ABC")
-   `SourceRunIDs` (TEXT, comma-separated list of `CTX_RunOutputID`s used to generate this composite)

### `CTX_CompositeDataPoints`
Stores the actual data points for a composite CTX curve.

-   `CompositeDataPointID` (Primary Key, e.g., INT AUTO_INCREMENT)
-   `CompositeCTX_ID` (INT, Foreign Key to `CTX_CompositeCurves.CompositeCTX_ID`)
-   `FrequencySeparation_MHz` (DECIMAL)
-   `WorstCaseObjectiveValue` (DECIMAL, the MIN value from source runs for this frequency)

## IV. Supporting/Contextual Tables (from MICS Reference - Book 2)

These tables are less directly involved in the CTXDOATD calculations described in Book 1 but are essential for a full MICS-like system for frequency coordination.

### `Sites` (`mt_site` / `me_site`)
-   `SiteID` (Primary Key)
-   `CallSign` (VARCHAR, UNIQUE for TS) / `LocationCode` (VARCHAR, UNIQUE for ES)
-   `SiteName` (VARCHAR)
-   `Latitude` (DECIMAL or specific format)
-   `Longitude` (DECIMAL or specific format)
-   `GroundElevation_m` (DECIMAL)
-   `CountryCode` (VARCHAR)
-   `OperatorCode` (VARCHAR, FK to `OperatingCompanies.OperatorCode`)

### `Links` (Implicit, derived from antenna pointing in TS or ES service arcs)
-   Potentially a table linking two `SiteID`s with path-specific data if needed beyond antenna records.

### `AntennaDeployments` (`mt_ante` / `me_ante`)
(How antennas are used at sites)
-   `DeployedAntennaID` (Primary Key)
-   `SiteID` (FK)
-   `AntennaCode` (FK to `Antennas.AntennaCode`)
-   `AntennaNumber_MICS` (INT, MICS specific usage)
-   `RemoteSiteCallSign` (VARCHAR, for TS, defines the link direction)
-   `Azimuth_deg` (DECIMAL)
-   `ElevationAngle_deg` (DECIMAL)
-   `AntennaHeightAGL_m` (DECIMAL)
-   `FeederLoss_Tx_dB` (DECIMAL)
-   `FeederLoss_Rx_dB` (DECIMAL)
-   `ServiceArcMidPointLongitude_deg` (DECIMAL, for ES)
-   `ServiceArcVariance_deg` (DECIMAL, for ES)

### `RadioChannels` (`mt_chan` / `me_chan`)
-   `Channel_MICS_ID` (Primary Key)
-   `DeployedAntennaID_Tx` (FK)
-   `DeployedAntennaID_Rx` (FK, could be same for TR antennas)
-   `SiteCallSign_Local` (VARCHAR, FK)
-   `SiteCallSign_Remote` (VARCHAR, FK, for TS)
-   `FrequencyBandCode_MICS` (VARCHAR, FK)
-   `ChannelIdentifier_MICS` (VARCHAR)
-   `FrequencyTx_MHz` (DECIMAL)
-   `FrequencyRx_MHz` (DECIMAL)
-   `EquipmentCode_Tx` (VARCHAR, FK to `EquipmentMasters.EquipmentCode`)
-   `EquipmentCode_Rx` (VARCHAR, FK to `EquipmentMasters.EquipmentCode`)
-   `TrafficCode` (VARCHAR, FK to `TrafficTypes.TrafficCode`)
-   `TransmitPower_dBm` (DECIMAL)
-   `ReceiveSignalLevel_Nominal_dBm` (DECIMAL)
-   `Polarization` (VARCHAR)
-   `Status` (VARCHAR, e.g. Planned, Operational)

## Rationale & Relationships

*   **Normalization & Centralization:** Master data for equipment, antennas, and traffic types are stored centrally to reduce redundancy and ensure consistency.
*   **Run-Specific vs. Master Data:** It's crucial to distinguish between the master characteristics of an equipment (`EquipmentMasters`) and the specific parameters used in a particular CTXDOATD run (`VictimDatasetRuns`, `InterfererDatasetRuns`). The latter allows for variations and exact recording of inputs for each calculation.
*   **Dual Conversion Handling:** The structure supports the multi-run approach by allowing multiple `CTX_RunOutputs` (each with a specific `IF_Used_For_Run_MHz`) to be associated with a single piece of (dual-conversion) victim equipment. These individual run outputs can then be aggregated into a `CTX_CompositeCurves`.
*   **Traceability:** The design aims to allow traceability from a final composite curve back to the individual runs that produced it, and from those runs back to the specific dataset parameters and master equipment records used.
*   **MICS Alignment:** Where Book 2 provided clear MICS table structures (like `sd_eqpt`, `sd_ante`, `sd_ctx`), the proposed tables try to align in spirit, though column names and specific normalizations might differ based on standard RDBMS practices vs. a legacy system.

This structure provides a robust way to manage the data for the described interference analysis, especially catering to the specific needs of analyzing dual conversion receivers using an iterative approach with a tool like CTXDOATD.
