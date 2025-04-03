# SBB-Passagierzahlen-Puenktlichkeit

## Datenaufbereitung des Datensatzes Pünktlichkeit

Der Datensatz für die Pünktlichkeit der Züge kann von der frei verfübaren Datenbank opentransportdata.swiss unter folgendem Link heruntergeladen werden: https://data.opentransportdata.swiss/dataset/istdaten. Für dieses Projekt wurden die Daten vom 27.3.2025 verwendet. 

Ein erster Überblick über die Daten erhalte ich mit Python und der Bibliothek Pandas:

```python
import pandas as pd

file_path_raw = "--pfad/2025-03-27_istdaten.csv"

df_raw = pd.read_csv(file_path_raw, delimiter=';')

# Spaltennamen abrufen
spalten = df_raw.columns.tolist()

# Anzahl der Zeilen abrufen
anzahl_zeilen = len(df_raw)

# Ergebnisse ausgeben
print("Spaltennamen:", spalten)
print("Gesamtanzahl der Zeilen:", anzahl_zeilen)
```

Der Datensatz beinhaltet 21 Variablen und 2'511'089 Einträge. Die für uns relevanten Variablen sind: BETREIBER_ABK (Abkürzung der Namen der Verkehrsbetriebe), BETREIBER_NAME (Name der Verkehrsbetriebe), BPUIC (Kenncode der Haltestellen), HALTESTELLEN_NAME (Name der Haltestellen), ANKUNFTSZEIT (Ankunftszeit gemäss Fahrplan), AN_PROGNOSE (tatsächliche Ankunftszeit), ABFAHRTSZEIT (Abfahrtszeit gemäss Fahrplan) und AB_PROGNOSE (tatsächliche Abfahrtszeit). Wenn wir die eindeutigen Einträge der Variable BETREIBER_NAME anschauen, sehen wir, dass neben der SBB auch noch alle regionalen Verkehrsbetriebe im Datensatz enthalten sind. 

```python
# Eindeutige Werte der Spalte BETREIBER_NAME
eindeutige_werte = df_raw['BETREIBER_NAME'].unique()

print(eindeutige_werte)
```
In einem ersten Schritt sollen alle Einträge von Verkehrsteilnehmer, die nicht SBB sind, herausgefiltert werden. Ausserdem sollen Einträge, welche in den relevanten Variablen ANKUNFTSZEIT (tatsächliche Ankunftszeit), AN_PROGNOSE (Ankunftszeit gemäss Fahrplan), ABFAHRTSZEIT (tatsächliche Abfahrtszeit), AB_PROGNOSE (Abfahrtszeit gemäss Fahrplan) keine Werte aufweisen auch herausgefiltert werden. 

```python
# CSV-Datei in Chunks verarbeiten
chunksize = 500000  # Anzahl Zeilen pro Chunk
filtered_chunks = []

# Direkter Pfad zur Datei
file_path = "--pfad/2025-03-27_istdaten.csv"
output_path = "--pfad/2025-03-27_istdaten_gefiltert.csv"

for chunk in pd.read_csv(file_path, chunksize=chunksize, delimiter=';'):
    # Anwenden der Filterbedingungen
    filtered_chunk = chunk[
        (chunk["BETREIBER_ABK"] == "SBB") &
        (chunk["BETREIBER_NAME"] == "Schweizerische Bundesbahnen SBB") &
        (chunk["ANKUNFTSZEIT"].notna()) &
        (chunk["AN_PROGNOSE"].notna()) &
        (chunk["ABFAHRTSZEIT"].notna()) &
        (chunk["AB_PROGNOSE"].notna())
    ]
    
    # Falls der Chunk nicht leer ist, speichern
    if not filtered_chunk.empty:
        filtered_chunks.append(filtered_chunk)

# Gefilterte Daten in neue CSV speichern
if filtered_chunks:
    pd.concat(filtered_chunks).to_csv(output_path, index=False)
else:
    print("Keine passenden Daten gefunden.")
```

Jetzt ist der Datensatz bereit, um in Power BI eingelesen zu werden. Anschliessend muss aus den Variablen ANKUNFTSZEIT und AN_PROGNOSE als auch aus den Variablen ABFAHRTSZEIT und AB_PROGNOSE die Differenz gebildet werden, damit die Pünktlichkeit des jeweiligen Zuges bestimmmt werden kann. 

```
= Table.AddColumn(
    #"Gefilterte Zeilen",
    "DauerBerechnung_Ankunft",
    each try [AN_PROGNOSE] - [ANKUNFTSZEIT] otherwise null,
    Duration.Type
 )
```

```
= Table.AddColumn(
    #"Ankunft_Differenz",
    "DauerBerechnung_Abfahrt",
    each try [AB_PROGNOSE] - [ABFAHRTSZEIT] otherwise null,
    Duration.Type
 )
```

Anschliessend kann aus den neu gebildeten Variablen zwei neue Variablen erstellt werden, welche die Pünktlichkeit als Kategorien speichern.

```
= Table.AddColumn(Abfahrt_Differenz, "Verspaetung_Ankunft_Kategorie", each if [DauerBerechnung_Ankunft] <= #duration(0, 0, 0, 0) then "A: Zu frühe/pünktliche Ankunft"
else if [DauerBerechnung_Ankunft] <= #duration(0, 0, 0, 29) then "B: < 30 Sekunden Verspätung"
else if [DauerBerechnung_Ankunft] <= #duration(0, 0, 0, 59) then "C: 30 Sekunden - 1 Minute Verspätung"
else if [DauerBerechnung_Ankunft] <= #duration(0, 0, 10, 0) then "D: 1-10 Minuten"
else if [DauerBerechnung_Ankunft] <= #duration(0, 0, 30, 0) then "E: 11-30 Minuten"
else "F: > 30 Minuten")
```

```
= Table.AddColumn(Verspaetung_Ankunft_Kategorie, "Verspaetung_Abfahrt_Kategorie", 
each if [DauerBerechnung_Abfahrt] <= #duration(0, 0, 0, 0) then "A: Zu frühe/pünktliche Abfahrt"
else if [DauerBerechnung_Abfahrt] <= #duration(0, 0, 0, 29) then "B: < 30 Sekunden Verspätung"
else if [DauerBerechnung_Abfahrt] <= #duration(0, 0, 0, 59) then "C: 30 Sekunden - 1 Minute Verspätung"
else if [DauerBerechnung_Abfahrt] <= #duration(0, 0, 10, 0) then "D: 1-10 Minuten"
else if [DauerBerechnung_Abfahrt] <= #duration(0, 0, 30, 0) then "E: 11-30 Minuten"
else "F: > 30 Minuten")
```

Dadurch gibt es jetzt im Datensatz die beiden Variablen "Verspaetung_Ankunft_Kategorie" und "Verspaetung_Abfahrt_Kategorie", welche angeben, ob ein Zug am 27.3.2025 zu früh oder pünktlich im Bahnhof angekommen oder abgefahren ist, oder ob er eine Verspätung von weniger als 30 Sekunden, zwischen 30 Sekunden und 1 Minute, zwischen 1-10 Minuten, zwischen 11-30 Minuten oder mehr als 30 Minuten aufweist. 



## Datenaufbereitung des Datensatzes Passagierzahlen

Die Passagierzahlen können auch von der frei verfübaren Datenbank opentransportdata.swiss heruntergladen werden: https://data.opentransportdata.swiss/dataset/einundaus.

Ein Überblick zeigt, dass es im Datensatz 14 Variablen hat, wobei UIC (Kennzahl Haltestellen), Jahn_Annee_Anno (Jahr der Erhebung) und DTV_TJM_TGM (Durchschnittlicher täglicher Verkehr (Montag bis Sonntag)) relevant sind. 

```python
file_path_passagiere = "--pfad/t01x-sbb-cff-ffs-frequentia-2023.xlsx"

df_passagiere = pd.read_excel(file_path_passagiere, sheet_name="Data")
                              
# Spaltennamen abrufen
spalten_passagiere = df_passagiere.columns.tolist()

# Anzahl der Zeilen abrufen
anzahl_zeilen_passagiere = len(df_passagiere)

# Ergebnisse ausgeben
print("Spaltennamen:", spalten_passagiere)
print("Gesamtanzahl der Zeilen:", anzahl_zeilen_passagiere)
```


## Zusammenführen der beiden Datensätze

Damit der Pünktlichkeits-Datensatz und der Passagier-Datensatz in Power BI als Datenmodell miteinander verknüpft werden können, muss ein Dimensions-Datensatz erstellt werden, welcher als Verbindungspunkt beider Datensätze fungiert. 

```python
# Pfade definieren
input_path = "--pfad/2025-03-27_istdaten_gefiltert.csv"
# Neuer Name für die Ausgabedatei mit den eindeutigen Bahnhöfen
output_path = "--pfad/eindeutige_bahnhoefe.csv" 

# Relevante Spalten definieren
columns_to_keep = ['BPUIC', 'HALTESTELLEN_NAME']

print(f"Lese Datei: {input_path}")
print(f"Wähle Spalten aus: {', '.join(columns_to_keep)}")

try:
    # Lese die gefilterte CSV-Datei ein, aber nur die benötigten Spalten
    df = pd.read_csv(input_path, delimiter=',', usecols=columns_to_keep)

    print(f"Anzahl Zeilen vor Deduplizierung: {len(df)}")

    # Entferne doppelte Zeilen basierend auf den ausgewählten Spalten
    df_unique = df.drop_duplicates()

    print(f"Anzahl Zeilen nach Deduplizierung: {len(df_unique)}")

    # Speichere das Ergebnis in einer neuen CSV-Datei
    df_unique.to_csv(output_path, index=False, sep=';') 

    print(f"Eindeutige Bahnhofsdaten gespeichert in: {output_path}")

except FileNotFoundError:
    print(f"FEHLER: Eingabedatei nicht gefunden unter {input_path}")
except KeyError as e:
    print(f"FEHLER: Spalte {e} nicht in der Datei {input_path} gefunden. Überprüfe die Spaltennamen.")
except Exception as e:
    print(f"Ein unerwarteter Fehler ist aufgetreten: {e}")
```

Dadurch wurde ein Datensatz erstellt, der jeden Bahnhof mit seiner zugehörigen Kennzahl enthält. In Power BI können die Datensätze als Datenmodell verbunden werden:

![grafik](https://github.com/user-attachments/assets/f6433cc4-4ae1-4541-8cb7-34ff9437b6eb)

![grafik](https://github.com/user-attachments/assets/f1c12e27-b134-4237-a200-de89845277c8)


Anschliessend kann ich das Dashboard erstellen:

![grafik](https://github.com/user-attachments/assets/256748b0-c411-41b5-afd8-e6b5d71f0309)




