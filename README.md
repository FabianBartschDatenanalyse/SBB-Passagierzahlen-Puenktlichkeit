# SBB-Passagierzahlen-Puenktlichkeit

## Datenaufbereitung

Die Passagierzahlen habe ich von der frei verfübaren Datenbank opentransportdata.swiss heruntergladen: https://data.opentransportdata.swiss/dataset/einundaus.

Der Datensatz für die Pönktlichkeit der Züge kann von derselben Webseite unter folgendem Link heruntergeladen werden: https://data.opentransportdata.swiss/dataset/istdaten. Für dieses Projekt wurden die Daten vom 27.3.2025 verwendet.
Dieser Datensatz verfügt über mehr als 2 Millionen Einträge von verschiedenen Verkehrsunternehmen. In einem ersten Schritt sollen alle Einträge von Verkehrsteilnehmer, die nicht SBB sind, herausgefiltert werden. Ausserdem sollen Einträge, welche in den relevanten Variablen ANKUNFTSZEIT (tatsächliche Ankunftszeit), AN_PROGNOSE (Ankunftszeit gemäss Fahrplan), ABFAHRTSZEIT (tatsächliche Abfahrtszeit), AB_PROGNOSE (Abfahrtszeit gemäss Fahrplan) keine Werte aufweisen auch herausgefiltert werden. 

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

```powerquer
= Table.AddColumn(
    #"Gefilterte Zeilen", // <-- Ersetze dies!
    "DauerBerechnung_Ankunft",              // <-- Name deiner neuen Spalte
    each try [AN_PROGNOSE] - [ANKUNFTSZEIT] otherwise null, // <-- Ersetze Spaltennamen!
    Duration.Type                   // Setzt den Typ der neuen Spalte auf Dauer
 )
```



