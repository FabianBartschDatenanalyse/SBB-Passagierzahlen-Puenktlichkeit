# SBB-Passagierzahlen-Puenktlichkeit

## Datenaufbereitung des Datensatzes Pünktlichkeit

Der Datensatz für die Pünktlichkeit der Züge kann von der frei verfübaren Datenbank opentransportdata.swiss unter folgendem Link heruntergeladen werden: https://data.opentransportdata.swiss/dataset/istdaten. Für dieses Projekt wurden die Daten vom 27.3.2025 verwendet. Dieser Datensatz verfügt über mehr als 2 Millionen Einträge von verschiedenen Verkehrsunternehmen. In einem ersten Schritt sollen alle Einträge von Verkehrsteilnehmer, die nicht SBB sind, herausgefiltert werden. Ausserdem sollen Einträge, welche in den relevanten Variablen ANKUNFTSZEIT (tatsächliche Ankunftszeit), AN_PROGNOSE (Ankunftszeit gemäss Fahrplan), ABFAHRTSZEIT (tatsächliche Abfahrtszeit), AB_PROGNOSE (Abfahrtszeit gemäss Fahrplan) keine Werte aufweisen auch herausgefiltert werden. 

```python
import pandas as pd

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

Die Passagierzahlen habe ich von der frei verfübaren Datenbank opentransportdata.swiss heruntergladen: https://data.opentransportdata.swiss/dataset/einundaus.
