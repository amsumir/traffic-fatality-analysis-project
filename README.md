# FARS Traffic Fatality Data: Merging & Cleaning Pipeline (2020–2023)

## Project Overview

This project ingests, merges, and cleans **Fatality Analysis Reporting System (FARS)** data from 2020–2023 for multi-year traffic fatality analysis. The pipeline combines 10 distinct FARS data tables across four years (40 individual files total) into 10 cleaned master datasets ready for regression modeling, hypothesis testing, and visualization.

---

## Table of Contents

1. [Data Extraction Process](#data-extraction-process)
2. [Data Merging (Step 1)](#data-merging-step-1)
3. [Data Cleaning (Step 2)](#data-cleaning-step-2)
4. [File Organization](#file-organization)
5. [Key Variables & Recoding](#key-variables--recoding)
6. [Data Quality Considerations](#data-quality-considerations)
7. [Next Steps](#next-steps)

---

## Data Extraction Process

### What is FARS?

The **Fatality Analysis Reporting System (FARS)** is a nationwide census maintained by the National Highway Traffic Safety Administration (NHTSA). FARS contains detailed information on all fatal traffic crashes (crashes resulting in at least one death) reported in the United States. The system has been collecting data since 1975.

**Source:** U.S. Department of Transportation / NHTSA National Center for Statistics and Analysis (NCSA)  
**Website:** [https://www.nhtsa.gov/research-data/fatality-analysis-reporting-system-fars](https://www.nhtsa.gov/research-data/fatality-analysis-reporting-system-fars)

### Data Collection Methodology

FARS data are obtained from existing state documents, including:

- Police crash reports
- Crash report supplements
- State vehicle registration files
- State driver records
- State roadway classification data
- Death certificates
- Toxicology reports
- Emergency Medical Service reports

**Trained state FARS analysts** code more than 170 data elements from these source documents into a standardized format. FARS data contains **no personal identifying information** (names, addresses, SSNs) and comply with the Privacy Act.

### How to Access FARS Data

FARS data for 2020–2023 can be accessed through:

1. **FARS Encyclopedia (Web Query):** [https://www-fars.nhtsa.dot.gov/](https://www-fars.nhtsa.dot.gov/)
   - Allows interactive queries and custom data runs
   - Suitable for targeted analyses

2. **FTP Download:** [ftp://ftp.nhtsa.dot.gov/FARS](ftp://ftp.nhtsa.dot.gov/FARS)
   - Download complete annual files (all states, all tables)
   - Files available in SAS, DBF, and CSV formats
   - Comprehensive for longitudinal analysis

3. **Data.gov Catalog:** [https://catalog.data.gov/](https://catalog.data.gov/)
   - Searchable FARS datasets by year
   - Includes auxiliary derived variables

### Data Download for This Project

For this project, complete FARS data for **2020, 2021, 2022, and 2023** were downloaded from the NHTSA FTP server in CSV format. Each year includes 10 separate table files:

| Table Name | Description | Level |
|-----------|-------------|-------|
| **accident** | Crash characteristics, time, location, environmental factors | Crash |
| **vehicle** | Vehicle information, speed, damage, rollover | Vehicle |
| **person** | Occupant demographics, injury severity, restraint use, test results | Person |
| **drugs** | Toxicology results, drug types, concentrations | Person |
| **race** | Race and ethnicity coding for occupants | Person |
| **damage** | Vehicle damage locations (clock position) | Vehicle |
| **driverrf** | Driver-related factors (e.g., speeding, overcorrecting) | Vehicle |
| **crashrf** | Crash-related factors (environmental/circumstantial) | Crash |
| **weather** | Weather conditions at time of crash | Crash |
| **safetyeq** | Safety equipment for non-motorists | Person |

### Data Hierarchy & Relationships

FARS tables follow a hierarchical structure with join keys:
- Crash (STATE + ST_CASE)
   - Vehicle (STATE + ST_CASE + VEH_NO)
      - Person (STATE + ST_CASE + VEH_NO + PER_NO)
         - Race (STATE + ST_CASE + VEH_NO + PER_NO)
         - Drugs (STATE + ST_CASE + VEH_NO + PER_NO)
         - Safety Equipment (STATE + ST_CASE + VEH_NO + PER_NO)
      - Damage (STATE + ST_CASE + VEH_NO)
      - Driver-Related Factors (STATE + ST_CASE + VEH_NO)
   - Crash-Related Factors (STATE + ST_CASE)
   - Weather (STATE + ST_CASE)


---

## Data Merging (Step 1)

### Objective

Combine identical table types across 2020–2023 into single master datasets (vertical merging / row-wise appending). This creates 10 master files, each containing 4 years of data with an added `YEAR` column indicating the source year.

### Input Files

- 40 CSV files organized by year:
  - `2020/accident.csv`, `2020/vehicle.csv`, ..., `2020/weather.csv`
  - `2021/accident.csv`, `2021/vehicle.csv`, ..., `2021/weather.csv`
  - `2022/...`, `2023/...`

### Process

Using R, a loop iterates through each table type (accident, vehicle, person, ..., weather) and each year (2020–2023):

1. **Read yearly file:** Load CSV into memory (e.g., `2020/accident.csv`)
2. **Store in list:** Accumulate all years' data in a named list
3. **Combine rows:** Use `do.call(rbind, yearly_data)` to stack rows vertically
4. **Add YEAR column:** Before stacking, prepend a `YEAR` variable to each yearly dataframe
5. **Reset row names:** Ensure clean sequential indexing
6. **Export master:** Write merged result to `accident_master_2020_2023.csv`, etc.

### Output

**10 master CSV files** saved to `master_files/` folder:

- `accident_master_2020_2023.csv` — 142,000+ rows
- `vehicle_master_2020_2023.csv`
- `person_master_2020_2023.csv`
- `drugs_master_2020_2023.csv`
- `damage_master_2020_2023.csv`
- `driverrf_master_2020_2023.csv`
- `weather_master_2020_2023.csv`
- `race_master_2020_2023.csv`
- `safetyeq_master_2020_2023.csv`
- `crashrf_master_2020_2023.csv`

**Key Column Additions:**
- `YEAR` — integer (2020, 2021, 2022, or 2023) indicating source year

---

## Data Cleaning (Step 2)

### Objective

Standardize data types, recode FARS-specific missing-value codes to proper R `NA` values, and create derived variables for analysis. This ensures consistency across tables and prepares data for regression modeling.

### FARS Missing-Value Coding

FARS uses specific numeric codes to denote unknown/not applicable/not tested values. These vary by variable but commonly include:

| Code Pattern | Meaning | Example Variables |
|-------------|---------|-------------------|
| `8, 9` | Unknown / Not reported | SEX, INJ_SEV, many binary codes |
| `88, 89, 98, 99` | Unknown / Not applicable | WEATHER, LGT_COND, REST_USE |
| `998, 999` | Test not given / Unknown | AGE, TRAV_SP, drug concentrations |
| `9999` | Unknown (high-value fields) | Some specialized codes |

**Problem:** These are technically valid numeric values, not true missing data, and can bias analyses if not recoded.

**Solution:** Replace these codes with R's `NA` to mark as missing.

### Cleaning Steps Implemented

#### Step 1: Standardize Key IDs and Types

Convert join keys to consistent integer types across all tables:
accident: STATE, ST_CASE, YEAR → integer
person: STATE, ST_CASE, VEH_NO, PER_NO, YEAR → integer
vehicle: STATE, ST_CASE, VEH_NO, YEAR → integer
drugs: STATE, ST_CASE, VEH_NO, PER_NO, YEAR → integer
... (and so on for all 10 tables)


**Rationale:** Ensures accurate merging and avoids type coercion errors.

#### Step 2: Recode Missing-Value Codes to NA

Applied table-specific recoding using helper functions:

- **make_na_89()** → Convert 8, 9 to NA
- **make_na_8899()** → Convert 88, 89, 98, 99 to NA
- **make_na_998999()** → Convert 998, 999, and higher to NA

**Examples:**

***Accident table***
accident$WEATHER <- make_na_8899(accident$WEATHER)
accident$LGT_COND <- make_na_8899(accident$LGT_COND)

***Person table***
person$INJ_SEV <- make_na_89(person$INJ_SEV)
person$REST_USE <- make_na_8899(person$REST_USE)
person$AGE <- make_na_998999(person$AGE)

***Drugs table***
drugs$DRUGRES <- make_na_998999(drugs$DRUGRES)

... (applied across all 10 tables)


#### Step 3: Create Derived Variables

Constructed analysis-ready indicators from cleaned variables:

**Person Table:**
- `IS_DRIVER` — Binary flag: 1 = driver, 0 = passenger/non-occupant
- `AGE_GRP` — Categorical: 0-15, 16-20, 21-25, 26-35, 36-50, 51-65, 66+

**Accident Table:**
- `NIGHT` — Binary flag: 1 = dark/low-light condition, 0 = daylight

**Vehicle Table:**
- `HIGH_SPEED` — Binary flag: 1 = traveled ≥70 mph, 0 = lower speed

**Drugs Table:**
- `ANY_DRUG` — Binary flag: 1 = any drug detected, 0 = no drug

**Driver-Related Factors Table:**
- `HAS_DRIVER_FACTOR` — Binary flag: 1 = at least one factor recorded, 0 = none

**Weather Table:**
- `ADVERSE_WEATHER` — Binary flag: 1 = rain/snow/fog/adverse condition, 0 = clear/good

**Safety Equipment Table:**
- `HAS_SAFETY_EQUIP` — Binary flag: 1 = safety equipment present, 0 = none

**Crash-Related Factors Table:**
- `HAS_CRASH_FACTOR` — Binary flag: 1 = crash factor recorded, 0 = none

**Rationale:** These derived variables align with research hypotheses (age groups, night vs. day, high speed, drug presence, driver behavioral factors) and simplify downstream modeling.

### Output

**10 cleaned master CSV files** saved to `cleaned/` folder:

- `accident_clean_2020_2023.csv`
- `vehicle_clean_2020_2023.csv`
- `person_clean_2020_2023.csv`
- `drugs_clean_2020_2023.csv`
- `damage_clean_2020_2023.csv`
- `driverrf_clean_2020_2023.csv`
- `weather_clean_2020_2023.csv`
- `race_clean_2020_2023.csv`
- `safetyeq_clean_2020_2023.csv`
- `crashrf_clean_2020_2023.csv`

**Key Additions:**
- FARS missing codes → proper `NA` values
- Derived binary/categorical indicators for modeling

---

## File Organization



### Data Flow
Raw Files (2020–2023)
↓ [Step 1: 01_vertical_merge.R]
Master Tables (merged years)
↓ [Step 2: 02_clean_fars.R]
Cleaned Tables (standardized, recoded, derived variables)
↓ [Step 3: 03_horizontal_merge.R]
Analysis Datasets (joined for modeling)


---

## Key Variables & Recoding

### Primary Join Keys (All Tables)

| Key | Type | Description |
|-----|------|-------------|
| STATE | integer | State FIPS code (01–56) |
| ST_CASE | integer | State + case number (unique crash ID within state-year) |
| VEH_NO | integer | Vehicle number within crash (1–9) |
| PER_NO | integer | Person number within vehicle (1–9) |
| YEAR | integer | 2020, 2021, 2022, or 2023 |

### Important Cleaned Variables by Table

#### Accident (Crash-Level)

| Variable | Type | Values | Notes |
|----------|------|--------|-------|
| WEATHER | numeric | 1–7 (after recoding) | 1=clear, 2=rain, 3=sleet/hail, etc. |
| LGT_COND | numeric | 1–9 (after recoding) | 1=daylight, 2–4=dark, 5–9=other |
| NIGHT | binary | 0 or 1 | 1 if LGT_COND in (2,3,4) |
| RELJCT1 | numeric | Relationship to junction | 1=intersection, 2=non-intersection, etc. |

#### Person (Occupant-Level)

| Variable | Type | Values | Notes |
|----------|------|--------|-------|
| INJ_SEV | numeric | 1–5 (after recoding) | 1=no injury, 5=fatal |
| REST_USE | numeric | 0–3 (after recoding) | 0=none, 1=seat belt, 2=shoulder/lap, 3=shoulder belt only |
| SEX | numeric | 1–2 (after recoding) | 1=male, 2=female |
| AGE | numeric | 0–120 (after recoding) | Age in years |
| AGE_GRP | factor | "0-15", "16-20", ..., "66+" | Categorical age group |
| IS_DRIVER | binary | 0 or 1 | 1 if PER_TYP == 1 |

#### Vehicle (Vehicle-Level)

| Variable | Type | Values | Notes |
|----------|------|--------|-------|
| TRAV_SP | numeric | 0–150 (after recoding) | Speed in mph |
| HIGH_SPEED | binary | 0 or 1 | 1 if TRAV_SP >= 70 |
| ROLLOVER | numeric | 0–2 | Vehicle rollover indicator |

#### Drugs (Person-Level)

| Variable | Type | Values | Notes |
|----------|------|--------|-------|
| DRUGRES | numeric | Drug codes (after recoding) | 3001–3999 = specific drugs |
| ANY_DRUG | binary | 0 or 1 | 1 if DRUGRES > 0 |
| DRUGTST | numeric | 0–7 (after recoding) | 0=not tested, 1=tested |

---

## Data Quality Considerations

### Known Issues & Mitigation

1. **Missing Values:** FARS uses specific numeric codes (8, 9, 88, 99, 998, 999) to represent missing/unknown/not applicable.
   - **Mitigation:** Recoded to `NA` using helper functions. Documented in cleaned files.

2. **Drug Testing Variation:** Not all states consistently test drivers/victims for drugs.
   - **Mitigation:** Track DRUGTST status. Filter to tested cases in analyses.

3. **Multiple Records per Key:** Some tables (drugs, damage) have multiple rows per person or vehicle.
   - **Rationale:** Preserved to maintain granularity; aggregate as needed for joins.

4. **Data Completeness:** Some jurisdictions report more detailed information than others.
   - **Rationale:** R's `NA` handling allows analysis despite incomplete data. Document response rates.

5. **Variable Changes Over Years:** FARS coding standards may shift between 2020–2023.
   - **Mitigation:** Refer to FARS Analytical User's Manual (1975–2023) for year-specific codebooks.

### Data Validation Steps Performed

✓ Row counts verified across merge steps  
✓ Join key uniqueness checked (STATE + ST_CASE + VEH_NO + PER_NO)  
✓ Year distribution confirmed (2020–2023 present)  
✓ Missing value codes identified and recoded  
✓ Derived variable logic spot-checked on sample rows  

---

## Next Steps

Once cleaned master tables are prepared:

1. **Horizontal Merging (Step 3):**
   - Join accident, person, vehicle, drugs tables to create analysis-ready datasets
   - One row per driver, one row per crash, etc., depending on research questions

2. **Exploratory Data Analysis:**
   - Descriptive statistics and distributions by year
   - Missing data patterns
   - Outlier identification and handling

3. **Hypothesis Testing & Modeling:**
   - Logistic regression: Model fatal injury probability
   - Ordinal regression: Model injury severity levels
   - Interaction effects: Alcohol + restraint use, speed + road type, etc.
   - Validation: Cross-validation, ROC curves, model diagnostics

4. **Visualization:**
   - Time series trends (2020–2023)
   - Geographic heat maps (by state)
   - Box plots: Injury severity by demographic groups
   - Interaction plots: Combined risk factors

---

## References

1. **NHTSA FARS:** https://www.nhtsa.gov/research-data/fatality-analysis-reporting-system-fars
2. **FARS Analytical User's Manual (1975–2023):** https://crashstats.nhtsa.dot.gov/
3. **FARS Coding and Validation Manual (2023):** https://crashstats.nhtsa.dot.gov/
4. **Data.gov FARS Catalog:** https://catalog.data.gov/ (search "FARS")
5. **NHTSA FTP Download:** ftp://ftp.nhtsa.dot.gov/FARS

---

## Contact & Questions

For questions about data extraction, merging, or cleaning, refer to:
- **NHTSA FARS Support:** https://www-fars.nhtsa.dot.gov/requests/datarequests.aspx
- **Project Documentation:** See `scripts/` folder for R code and inline comments

---

**Document Generated:** December 2, 2025  
**Data Years:** 2020–2023  
**Tables Included:** 10 (accident, vehicle, person, drugs, damage, driverrf, weather, race, safetyeq, crashrf)  
**Total Records:** 140,000+ (merged across 4 years)



