# Hackathon Challenge: Entwicklung des Radverkehrs in Mannheim und Baden-Württemberg

## Fragestellung

- In welchen Städten wächst der Radverkehr – und wo nicht?
- Hängt dies messbar mit Infrastrukturmaßnahmen oder dem Fahrradklima zusammen?

## Datengrundlage

### Primäre Datenquelle: MobiData BW – Eco-Counter

Die Hauptdatenquelle für diese Challenge sind die Fahrradzähldaten aus dem landesweiten
MobiData BW Portal, bereitgestellt über das Eco-Counter-Netzwerk:

[MobiData BW – Eco-Counter Fahrradzähler](https://mobidata-bw.de/dataset/eco-counter-fahrradzahler)

Die Daten umfassen automatische Fahrradzählstellen aus verschiedenen Städten in
Baden-Württemberg und erlauben einen zeitlichen sowie räumlichen Vergleich des Radverkehrs.

Ein Tutorial zum Datenabruf steht bereit unter:
[Tutorial EcoCounter-Daten (PDF)](https://mobidata-bw.de/data/Tutorial_EcoCounter-Daten.pdf)

## Aufgabenstellung

### 1. Datenexploration

- Lade und erkunde die Eco-Counter-Daten aus MobiData BW
- Identifiziere verfügbare Zählstellen, Zeiträume und Datenqualität

### 2. Trendanalyse

- Analysiere die Entwicklung des Radverkehrs über die Zeit (z. B. Jahresvergleiche,
  saisonale Muster)
- Identifiziere Städte oder Zählstellen mit deutlich wachsendem bzw. rückläufigem
  Radverkehr
- Vergleiche optional Rad- und MIV-Trend in Mannheim (Modal-Split-Entwicklung)

### 3. Korrelationsanalyse

Untersuche mögliche Zusammenhänge mit externen Faktoren:

- **Infrastrukturmaßnahmen**: Neu gebaute Radwege, Protected Bike Lanes,
  Fahrradstraßen (Quellen: z. B. OpenStreetMap-Historie, kommunale Pressemitteilungen)
- **Fahrradklima**: ADFC-Fahrradklima-Test
- **Weitere Faktoren**: Wetter, Bevölkerungsdichte, Modal-Split-Daten

### 4. Visualisierung & Präsentation

- Zeitreihen-Diagramme je Stadt/Zählstelle
- Karte der Zählstellen mit Trendindikator (wachsend / stagnierend / rückläufig)
- Gegenüberstellung: Radverkehrswachstum vs. Fahrradklima-Score

## Mögliche Zusatzdaten

| Datenquelle | Inhalt | Link |
|---|---|---|
| ADFC Fahrradklima-Test | Stadtbewertungen 2020–2024 | [fahrradklima-test.adfc.de](https://fahrradklima-test.adfc.de) |
| OpenStreetMap | Radinfrastruktur historisch | [openstreetmap.org](https://openstreetmap.org) |
| Statistisches Landesamt BW | Bevölkerungsdaten | [statistik-bw.de](https://daten.statistik-bw.de/genesisonline//online?operation=table&code=12111_0005&bypass=true&levelindex=1&levelid=1776522265740#abreadcrumb) |
| Mobilitätsportal Mannheim | Radwegenetz Mannheim | [gis-mannheim.de/mannheim_mobi](https://gis-mannheim.de/mannheim_mobi/) |
| RadNETZ Baden-Württemberg | Landesweites Radverkehrsnetz, Ausbauplanung | [aktivmobil-bw.de](https://www.aktivmobil-bw.de/radverkehr/radnetz/uebersicht-radnetz-baden-wuerttemberg) |
| Verkehrszähldaten BW | MIV-Dauerzählstellen, manuelle und temporäre Zählstellen landesweit | [mobidata-bw.de/group/verkehrszaehldaten](https://mobidata-bw.de/group/verkehrszaehldaten) |

## Tipps

- Für den Städtevergleich lohnt es sich, die Zählwerte auf die Bevölkerungszahl
  zu normalisieren
- Radzähler fallen gelegentlich aus oder liefern auffällig hohe oder niedrige Werte - solche Zeiträume vor der Analyse prüfen
- Saisonalität (Sommer vs. Winter) verzerrt Jahresvergleiche – ein gleitender
  Durchschnitt oder ein Vorjahresvergleich (YoY) ist robuster als ein einfacher Trend

---

## Ergänzende Datenquelle: OpenData Mannheim via VictoriaMetrics

Als Alternative zum direkten Abruf über das OpenData Portal der Stadt Mannheim steht
eine VictoriaMetrics-Instanz mit aufbereiteten Zähldaten zur Verfügung:

🔗 [VictoriaMetrics UI](https://monitor.quadradentscheid.de/victoriametrics/vmui/)

Verfügbare Metriken:

| Metrik | Inhalt | Quelle |
|---|---|---|
| `eco_bike_counters_value` | Fahrradzählstellen Mannheim (19 Standorte, ab 2014) | [OpenData Mannheim – Eco Counter](https://mannheim.opendatasoft.com/explore/?sort=modified&refine.publisher=Eco+Counter+f%C3%BCr+FB+Geoinformation+und+Stadtplanung) |
| `traffic_count` | MIV-Verkehrszählung Mannheim | [opendata.smartmannheim.de](https://opendata.smartmannheim.de/organization/smart-mannheim) |

### Datenabruf per Python

#### Abfrage der Daten von victoriametrics

```python
import requests
import pandas as pd

response = requests.get(
    "https://monitor.quadradentscheid.de/victoriametrics/api/v1/query_range",
    auth=("hackathon", "<passwort>"),
    params={
        "query": "sum by(name) (sum_over_time(eco_bike_counters_value[1d]))",
        "start": "2023-01-01T00:00:00Z",
        "end":   "2026-04-18T00:00:00Z",
        "step":  "1d"
    }
)

# Prometheus-Format in DataFrame umwandeln
results = response.json()["data"]["result"]
frames = []
for series in results:
    df = pd.DataFrame(series["values"], columns=["timestamp", "count"])
    df["timestamp"] = pd.to_datetime(df["timestamp"], unit="s")
    df["count"] = pd.to_numeric(df["count"])
    df["station"] = series["metric"].get("name", "unbekannt")
    frames.append(df)

combined_df = pd.concat(frames).set_index("timestamp")
```

#### Alle Daten in einem Graph anzeigen
```python
import matplotlib.pyplot as plt

combined_df.pivot_table(
    index=combined_df.index, columns="station", values="count"
).plot(figsize=(14, 6), title="Alle Zählstellen im Vergleich", ylabel="Fahrten")
plt.legend(bbox_to_anchor=(1.01, 1), loc="upper left")  # Legende außerhalb
plt.tight_layout()
plt.show()
```

> **Hinweis:** Die VictoriaMetrics-API gibt Zeitreihendaten im Prometheus-Format zurück –
> `timestamps` sind Unix-Sekunden und müssen mit `unit="s"` konvertiert werden.
> Die MIV-Zähldaten (`traffic_count`) eignen sich als Gegenpol zum Radverkehr:
> Sinkt der Autoverkehr, wo der Radverkehr wächst?

### Grafana-Dashboard

Für eine schnelle visuelle Exploration der Mannheimer Zähldaten steht ein
Grafana-Dashboard zur Verfügung:

🔗 [Grafana Dashboard](https://monitor.quadradentscheid.de/)

> **Hinweis:** Anmeldedaten für Grafana und die VictoriaMetrics-Instanz
> werden vor Ort bereitgestellt.

---

Viel Erfolg beim Hackathon! 🚲
