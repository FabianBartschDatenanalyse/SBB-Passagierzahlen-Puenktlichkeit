# SBB Passenger Numbers and Punctuality

## Data Preparation: Train Punctuality Dataset

The dataset for train punctuality can be downloaded from the open data platform [opentransportdata.swiss](https://data.opentransportdata.swiss/dataset/istdaten).  
For this project, the data from **March 27, 2025** was used.

We start with an initial overview using Python and the Pandas library:

```python
import pandas as pd

file_path_raw = "--path/2025-03-27_istdaten.csv"

df_raw = pd.read_csv(file_path_raw, delimiter=';')

# Get column names
columns = df_raw.columns.tolist()

# Get number of rows
row_count = len(df_raw)

# Output results
print("Column names:", columns)
print("Total number of rows:", row_count)
```

The dataset contains **21 variables** and **2,511,089 entries**. The relevant variables for this project are:

- `BETREIBER_ABK` (Operator abbreviation)  
- `BETREIBER_NAME` (Operator name)  
- `BPUIC` (Station code)  
- `HALTESTELLEN_NAME` (Station name)  
- `ANKUNFTSZEIT` (Scheduled arrival time)  
- `AN_PROGNOSE` (Actual arrival time)  
- `ABFAHRTSZEIT` (Scheduled departure time)  
- `AB_PROGNOSE` (Actual departure time)

Let’s look at the unique entries in the `BETREIBER_NAME` column to identify included operators:

```python
# Unique operator names
unique_values = df_raw['BETREIBER_NAME'].unique()

print(unique_values)
```

As a first step, we filter out entries that are **not** operated by SBB. We also remove rows where any of the key timing fields (`ANKUNFTSZEIT`, `AN_PROGNOSE`, `ABFAHRTSZEIT`, `AB_PROGNOSE`) are missing.

```python
# Process CSV in chunks
chunksize = 500000
filtered_chunks = []

file_path = "--path/2025-03-27_istdaten.csv"
output_path = "--path/2025-03-27_istdaten_filtered.csv"

for chunk in pd.read_csv(file_path, chunksize=chunksize, delimiter=';'):
    filtered_chunk = chunk[
        (chunk["BETREIBER_ABK"] == "SBB") &
        (chunk["BETREIBER_NAME"] == "Schweizerische Bundesbahnen SBB") &
        (chunk["ANKUNFTSZEIT"].notna()) &
        (chunk["AN_PROGNOSE"].notna()) &
        (chunk["ABFAHRTSZEIT"].notna()) &
        (chunk["AB_PROGNOSE"].notna())
    ]
    
    if not filtered_chunk.empty:
        filtered_chunks.append(filtered_chunk)

# Save filtered results
if filtered_chunks:
    pd.concat(filtered_chunks).to_csv(output_path, index=False)
else:
    print("No matching data found.")
```

The dataset is now ready to be imported into **Power BI**. Next, we calculate the time differences between actual and scheduled arrival/departure times.

```powerquery
= Table.AddColumn(
    #"Filtered Rows",
    "Duration_Arrival",
    each try [AN_PROGNOSE] - [ANKUNFTSZEIT] otherwise null,
    Duration.Type
)
```

```powerquery
= Table.AddColumn(
    #"Arrival Duration",
    "Duration_Departure",
    each try [AB_PROGNOSE] - [ABFAHRTSZEIT] otherwise null,
    Duration.Type
)
```

We then classify the delays into categories for easier analysis:

```powerquery
= Table.AddColumn(Departure_Duration, "Arrival_Delay_Category", 
each if [Duration_Arrival] <= #duration(0, 0, 0, 0) then "A: Early/on-time arrival"
else if [Duration_Arrival] <= #duration(0, 0, 0, 29) then "B: < 30 seconds delay"
else if [Duration_Arrival] <= #duration(0, 0, 0, 59) then "C: 30 sec – 1 min delay"
else if [Duration_Arrival] <= #duration(0, 0, 10, 0) then "D: 1–10 minutes"
else if [Duration_Arrival] <= #duration(0, 0, 30, 0) then "E: 11–30 minutes"
else "F: > 30 minutes")
```

```powerquery
= Table.AddColumn(Arrival_Delay_Category, "Departure_Delay_Category", 
each if [Duration_Departure] <= #duration(0, 0, 0, 0) then "A: Early/on-time departure"
else if [Duration_Departure] <= #duration(0, 0, 0, 29) then "B: < 30 seconds delay"
else if [Duration_Departure] <= #duration(0, 0, 0, 59) then "C: 30 sec – 1 min delay"
else if [Duration_Departure] <= #duration(0, 0, 10, 0) then "D: 1–10 minutes"
else if [Duration_Departure] <= #duration(0, 0, 30, 0) then "E: 11–30 minutes"
else "F: > 30 minutes")
```

Now, the dataset contains two additional columns — `Arrival_Delay_Category` and `Departure_Delay_Category` — which categorize whether a train arrived or departed on time, or with varying degrees of delay on **March 27, 2025**.

---

## Data Preparation: Passenger Numbers

Passenger statistics are available from the same open data platform:  
https://data.opentransportdata.swiss/dataset/einundaus

The dataset contains **14 variables**, but the relevant ones are:  
- `UIC` (Station identifier)  
- `Jahr_Annee_Anno` (Year of measurement)  
- `DTV_TJM_TGM` (Average daily passenger frequency)

```python
file_path_passengers = "--path/t01x-sbb-cff-ffs-frequentia-2023.xlsx"

df_passengers = pd.read_excel(file_path_passengers, sheet_name="Data")

# Retrieve column names and number of rows
columns_passengers = df_passengers.columns.tolist()
row_count_passengers = len(df_passengers)

print("Column names:", columns_passengers)
print("Total number of rows:", row_count_passengers)
```

---

## Merging the Two Datasets

To link the punctuality and passenger datasets in Power BI, we need a **dimension table** that can act as a bridge.

```python
input_path = "--path/2025-03-27_istdaten_filtered.csv"
output_path = "--path/unique_stations.csv"

columns_to_keep = ['BPUIC', 'HALTESTELLEN_NAME']

print(f"Reading file: {input_path}")
print(f"Selecting columns: {', '.join(columns_to_keep)}")

try:
    df = pd.read_csv(input_path, delimiter=',', usecols=columns_to_keep)

    print(f"Rows before deduplication: {len(df)}")

    df_unique = df.drop_duplicates()

    print(f"Rows after deduplication: {len(df_unique)}")

    df_unique.to_csv(output_path, index=False, sep=';')

    print(f"Unique station data saved to: {output_path}")

except FileNotFoundError:
    print(f"ERROR: Input file not found at {input_path}")
except KeyError as e:
    print(f"ERROR: Column {e} not found in {input_path}. Check column names.")
except Exception as e:
    print(f"An unexpected error occurred: {e}")
```

This creates a station reference dataset linking station names and codes. In **Power BI**, you can now create a data model to join the datasets:

![diagram](https://github.com/user-attachments/assets/f6433cc4-4ae1-4541-8cb7-34ff9437b6eb)

![diagram](https://github.com/user-attachments/assets/f1c12e27-b134-4237-a200-de89845277c8)

---

## Final Dashboard

Once the model is set up, you can create your final dashboard:

![dashboard](https://github.com/user-attachments/assets/256748b0-c411-41b5-afd8-e6b5d71f0309)
