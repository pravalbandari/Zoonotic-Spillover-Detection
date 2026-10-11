import json
import os
import urllib.request
import urllib.parse
from datetime import datetime
from flask import Flask, request, render_template_string

app = Flask(__name__)
DATA_FILE = "reports.json"

# ---------------------------------------------------------
# REFERENCE DATA (placeholder weights - have a vet or
# epidemiologist review these before real-world use)
# ---------------------------------------------------------
SPECIES_POINTS = {
    "bat": 25, "primate": 22, "rodent": 20, "poultry": 18,
    "pig": 18, "camel": 18, "cattle": 12, "dog": 10, "other": 8,
}
CONTACT_POINTS = {
    "hunting / bushmeat handling": 20, "slaughter": 18,
    "caves / forests": 18, "wet market": 16, "farming": 12, "pets": 6,
}
LEVEL_COLORS = {
    "LOW": "#2e7d32", "MEDIUM": "#f9a825",
    "HIGH": "#ef6c00", "CRITICAL": "#c62828",
}


# ---------------------------------------------------------
# LIVE DATA: current weather from Open-Meteo (free, no key)
# ---------------------------------------------------------
def get_live_weather(lat, lon):
    params = urllib.parse.urlencode({
        "latitude": lat,
        "longitude": lon,
        "current": "temperature_2m,relative_humidity_2m,precipitation",
    })
    url = "https://api.open-meteo.com/v1/forecast?" + params
    try:
        with urllib.request.urlopen(url, timeout=10) as response:
            data = json.loads(response.read().decode("utf-8"))
        return data["current"]
    except Exception as error:
        print("Weather fetch failed:", error)
        return None


# ---------------------------------------------------------
# RISK ENGINE
# ---------------------------------------------------------
def calculate_risk(report, weather):
    breakdown = []
    breakdown.append((f"Species: {report['species']}", SPECIES_POINTS[report["species"]]))
    breakdown.append((f"Contact type: {report['contact']}", CONTACT_POINTS[report["contact"]]))
    breakdown.append((f"Sick/dead animals: {report['animals_dead']}", min(report["animals_dead"] * 2, 20)))
    breakdown.append((f"People with symptoms: {report['human_cases']}", min(report["human_cases"] * 5, 20)))

    if not report["ppe_used"]:
        breakdown.append(("No protective equipment used", 10))

    if weather:
        temp = weather.get("temperature_2m")
        humidity = weather.get("relative_humidity_2m")
        rain = weather.get("precipitation")
        pts = 0
        if humidity is not None and humidity >= 80:
            pts += 3
        if rain is not None and rain > 0:
            pts += 2
        if temp is not None and 25 <= temp <= 35:
            pts += 2
        if pts > 0:
            breakdown.append(("Live weather conditions", pts))

    score = min(sum(p for _, p in breakdown), 100)
    if score < 30:
        level = "LOW"
    elif score < 55:
        level = "MEDIUM"
    elif score < 75:
        level = "HIGH"
    else:
        level = "CRITICAL"
    return score, level, breakdown


# ---------------------------------------------------------
# RECOMMENDATION ENGINE
# ---------------------------------------------------------
def get_recommendations(level, report):
    if level == "LOW":
        actions = [
            "Continue routine monitoring of the animals and people involved.",
            "Wash hands thoroughly after handling animals.",
            "Note any change in symptoms or animal health.",
        ]
    elif level == "MEDIUM":
        actions = [
            "Separate sick animals from healthy ones.",
            "Use gloves and a mask when handling animals or carcasses.",
            "Inform your local veterinarian and ask for an inspection.",
            "Monitor exposed people for fever or other symptoms for several days.",
        ]
    elif level == "HIGH":
        actions = [
            "Isolate affected animals and restrict movement in and out of the area.",
            "Contact local veterinary and public health authorities today.",
            "Do not handle carcasses without full protective equipment.",
            "Arrange sample collection through trained professionals.",
            "Exposed people should seek medical advice and report symptoms early.",
        ]
    else:
        actions = [
            "Treat as an emergency: notify public health and veterinary authorities immediately.",
            "Strictly isolate the site, animals, and exposed people as advised by authorities.",
            "Do not move, sell, or slaughter animals from the affected area.",
            "Only trained personnel with full PPE should handle animals or samples.",
            "Seek medical care for anyone with symptoms and mention the animal contact.",
        ]

    if not report["ppe_used"]:
        actions.append("Start using protective equipment (gloves, mask, boots) from now on.")
    if report["contact"] in ("hunting / bushmeat handling", "wet market"):
        actions.append("Avoid handling or eating meat from sick or dead animals.")
    if report["species"] == "bat":
        actions.append("Avoid contact with bat droppings, saliva, and partially eaten fruit.")
    if report["species"] in ("poultry", "pig"):
        actions.append("Clean and disinfect pens, tools, and vehicles; limit visitors to the farm.")
    return actions


# ---------------------------------------------------------
# STORAGE
# ---------------------------------------------------------
def save_report(report):
    with open(DATA_FILE, "a", encoding="utf-8") as f:
        f.write(json.dumps(report) + "\n")


def load_reports():
    reports = []
    if not os.path.exists(DATA_FILE):
        return reports
    with open(DATA_FILE, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if line:
                try:
                    reports.append(json.loads(line))
                except json.JSONDecodeError:
                    pass
    return reports


# ---------------------------------------------------------
# HTML TEMPLATES
# ---------------------------------------------------------
PAGE_TOP = """
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>SpilloverWatch</title>
<style>
  body { font-family: Arial, sans-serif; margin: 0; background: #f4f6f8; color: #222; }
  header { background: #1b4332; color: white; padding: 14px 24px; }
  header a { color: #d8f3dc; margin-right: 18px; text-decoration: none; font-weight: bold; }
  header h1 { margin: 0 0 8px 0; font-size: 22px; }
  main { max-width: 820px; margin: 24px auto; padding: 0 16px; }
  .card { background: white; border-radius: 8px; padding: 20px; margin-bottom: 18px;
          box-shadow: 0 1px 4px rgba(0,0,0,0.12); }
  label { display: block; margin-top: 12px; font-weight: bold; }
  input, select { width: 100%; padding: 9px; margin-top: 4px; box-sizing: border-box;
                  border: 1px solid #bbb; border-radius: 5px; font-size: 15px; }
  button { margin-top: 18px; background: #2d6a4f; color: white; border: 0;
           padding: 11px 22px; border-radius: 5px; font-size: 16px; cursor: pointer; }
  button:hover { background: #1b4332; }
  .error { background: #fde8e8; color: #9b1c1c; padding: 10px; border-radius: 5px; margin-bottom: 12px; }
  .badge { display: inline-block; color: white; padding: 6px 16px; border-radius: 20px;
           font-weight: bold; font-size: 18px; }
  .bar { background: #ddd; border-radius: 6px; height: 16px; margin: 10px 0; overflow: hidden; }
  .bar div { height: 100%; }
  table { width: 100%; border-collapse: collapse; font-size: 14px; }
  th, td { text-align: left; padding: 8px; border-bottom: 1px solid #e3e3e3; }
  .note { font-size: 13px; color: #555; }
  #map { height: 380px; border-radius: 8px; }
  .row { display: flex; gap: 12px; }
  .row > div { flex: 1; }
</style>
</head>
<body>
<header>
  <h1>SpilloverWatch</h1>
  <a href="/">New Report</a>
  <a href="/history">History &amp; Map</a>
</header>
<main>
"""

PAGE_BOTTOM = """
</main>
</body>
</html>
"""

FORM_BODY = """
<div class="card">
  <h2>Report an animal-human contact event</h2>
  {% if error %}<div class="error">{{ error }}</div>{% endif %}
  <form method="post">
    <label>Location name (village / town)</label>
    <input name="location_name" required value="{{ form.get('location_name', '') }}">

    <div class="row">
      <div>
        <label>Latitude (e.g. 12.97)</label>
        <input name="lat" required value="{{ form.get('lat', '') }}">
      </div>
      <div>
        <label>Longitude (e.g. 79.15)</label>
        <input name="lon" required value="{{ form.get('lon', '') }}">
      </div>
    </div>

    <label>Animal species involved</label>
    <select name="species">
      {% for s in species_list %}
        <option value="{{ s }}" {% if form.get('species') == s %}selected{% endif %}>{{ s }}</option>
      {% endfor %}
    </select>

    <label>Type of contact</label>
    <select name="contact">
      {% for c in contact_list %}
        <option value="{{ c }}" {% if form.get('contact') == c %}selected{% endif %}>{{ c }}</option>
      {% endfor %}
    </select>

    <div class="row">
      <div>
        <label>Number of sick / dead animals</label>
        <input name="animals_dead" type="number" min="0" value="{{ form.get('animals_dead', 0) }}" required>
      </div>
      <div>
        <label>Number of people with symptoms</label>
        <input name="human_cases" type="number" min="0" value="{{ form.get('human_cases', 0) }}" required>
      </div>
    </div>

    <label>Was protective equipment used?</label>
    <select name="ppe">
      <option value="yes" {% if form.get('ppe') == 'yes' %}selected{% endif %}>Yes</option>
      <option value="no" {% if form.get('ppe') == 'no' %}selected{% endif %}>No</option>
    </select>

    <button type="submit">Analyze risk</button>
  </form>
</div>
"""

RESULT_BODY = """
<div class="card">
  <h2>Result for {{ report.location_name }}</h2>
  <p class="note">{{ report.time_display }}</p>
  <span class="badge" style="background: {{ color }};">{{ level }} RISK</span>
  <p style="font-size: 20px; margin: 12px 0 0 0;"><b>{{ score }}</b> / 100</p>
  <div class="bar"><div style="width: {{ score }}%; background: {{ color }};"></div></div>
</div>

{% if weather %}
<div class="card">
  <h3>Live weather at this location</h3>
  <p>Temperature: <b>{{ weather.temperature_2m }} &deg;C</b> &nbsp;|&nbsp;
     Humidity: <b>{{ weather.relative_humidity_2m }} %</b> &nbsp;|&nbsp;
     Precipitation: <b>{{ weather.precipitation }} mm</b></p>
  <p class="note">Source: Open-Meteo</p>
</div>
{% else %}
<div class="card"><p class="note">Live weather could not be fetched, so it was not used in the score.</p></div>
{% endif %}

<div class="card">
  <h3>Why this score</h3>
  <table>
    {% for reason, pts in breakdown %}
      <tr><td>{{ reason }}</td><td>+{{ pts }}</td></tr>
    {% endfor %}
  </table>
</div>

<div class="card">
  <h3>Recommended actions</h3>
  <ol>
    {% for a in actions %}<li style="margin-bottom: 6px;">{{ a }}</li>{% endfor %}
  </ol>
</div>

<p class="note">This tool supports decisions only. It does not replace laboratory
confirmation or advice from health authorities.</p>
<a href="/"><button>Submit another report</button></a>
"""

HISTORY_BODY = """
<div class="card">
  <h2>Reported events map</h2>
  <div id="map"></div>
  <p class="note">Map uses OpenStreetMap (needs internet).</p>
</div>

<div class="card">
  <h2>Recent reports</h2>
  {% if reports %}
  <table>
    <tr><th>Time</th><th>Location</th><th>Species</th><th>Score</th><th>Level</th></tr>
    {% for r in reports %}
      <tr>
        <td>{{ r.time_display }}</td>
        <td>{{ r.location_name }}</td>
        <td>{{ r.species }}</td>
        <td>{{ r.score }}</td>
        <td><b style="color: {{ colors[r.level] }};">{{ r.level }}</b></td>
      </tr>
    {% endfor %}
  </table>
  {% else %}
    <p>No reports yet. Submit one from the New Report page.</p>
  {% endif %}
</div>

<link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">
<script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
<script>
  var reports = {{ reports|tojson }};
  var colors = {{ colors|tojson }};
  var map = L.map('map').setView([20, 78], 4);
  L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
    maxZoom: 18, attribution: '&copy; OpenStreetMap contributors'
  }).addTo(map);

  var points = [];
  reports.forEach(function(r) {
    var m = L.circleMarker([r.lat, r.lon], {
      radius: 9, color: colors[r.level], fillColor: colors[r.level], fillOpacity: 0.7
    }).addTo(map);
    m.bindPopup('<b>' + r.location_name + '</b><br>' + r.species +
                '<br>Risk: ' + r.score + ' (' + r.level + ')');
    points.push([r.lat, r.lon]);
  });
  if (points.length > 0) { map.fitBounds(points, {padding: [30, 30], maxZoom: 10}); }
</script>
"""


# ---------------------------------------------------------
# ROUTES
# ---------------------------------------------------------
@app.route("/", methods=["GET", "POST"])
def index():
    error = None
    if request.method == "POST":
        try:
            lat = float(request.form["lat"])
            lon = float(request.form["lon"])
            animals_dead = int(request.form["animals_dead"])
            human_cases = int(request.form["human_cases"])
            species = request.form["species"]
            contact = request.form["contact"]

            if not (-90 <= lat <= 90 and -180 <= lon <= 180):
                raise ValueError
            if animals_dead < 0 or human_cases < 0:
                raise ValueError
            if species not in SPECIES_POINTS or contact not in CONTACT_POINTS:
                raise ValueError
        except (ValueError, KeyError):
            error = ("Invalid input. Latitude must be between -90 and 90, longitude "
                     "between -180 and 180, and counts must be whole numbers (0 or more).")
        else:
            now = datetime.now()
            report = {
                "location_name": request.form["location_name"].strip(),
                "lat": lat,
                "lon": lon,
                "species": species,
                "contact": contact,
                "animals_dead": animals_dead,
                "human_cases": human_cases,
                "ppe_used": request.form.get("ppe") == "yes",
                "time": now.isoformat(),
                "time_display": now.strftime("%Y-%m-%d %H:%M"),
            }
            weather = get_live_weather(lat, lon)
            score, level, breakdown = calculate_risk(report, weather)
            actions = get_recommendations(level, report)

            report["score"] = score
            report["level"] = level
            report["weather"] = weather
            save_report(report)

            return render_template_string(
                PAGE_TOP + RESULT_BODY + PAGE_BOTTOM,
                report=report, weather=weather, score=score, level=level,
                color=LEVEL_COLORS[level], breakdown=breakdown, actions=actions,
            )

    return render_template_string(
        PAGE_TOP + FORM_BODY + PAGE_BOTTOM,
        error=error, form=request.form,
        species_list=list(SPECIES_POINTS.keys()),
        contact_list=list(CONTACT_POINTS.keys()),
    )


@app.route("/history")
def history():
    reports = load_reports()
    reports.reverse()  # newest first
    return render_template_string(
        PAGE_TOP + HISTORY_BODY + PAGE_BOTTOM,
        reports=reports[:100], colors=LEVEL_COLORS,
    )


if __name__ == "__main__":
    app.run(debug=False)
