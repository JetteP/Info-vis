# De invloed van klimaat en geografie op voedselproductie
Neerslag, temperatuur en klimaatzone bepalen welke gewassen vanzelf groeien en hoeveel energie een boer moet bijpompen. In warme, natte gebieden ontstaan hoge opbrengsten met weinig inputs; in koude of droge streken vergen irrigatie, kassen en extra voer juist veel uitstoot. Ook afstand telt: hoe verder een land van zijn afzet- of aanvoermarkt ligt, hoe meer transport-CO₂ het voedselsysteem meedraagt. Dit perspectief laat met vier uiterste voorbeelden zien hoe natuurlijke groeicondities én geografische ligging samen de klimaatafdruk van nationale landbouwsystemen vormen.
Binnen dit perspectief onderscheiden we twee kernargumenten:
- Internationale handelsstromen beïnvloeden nationale uitstoot
- De geografische ligging van een land (en de bijbehorende klimaatzones) beïnvloedt welke gewassen en dieren men produceert — en dat heeft directe impact op de landbouwgerelateerde CO₂-uitstoot per capita.


### Internationale handelsstromen beïnvloeden nationale uitstoot

Sinds de 19e eeuw hebben landen met gemakkelijk bereikbare zee- of rivierhavens een structureel transportvoordeel. Die geografische ligging vormde vroege exportclusters voor graan, vlees en oliegewassen, die in veel gevallen nog altijd dominant zijn. Daardoor gaan we ervan uit dat zulke “logistiek gunstige” landen tegenwoordig zowel grote landbouw­exportvolumes als een hogere landbouw-CO₂-uitstoot per inwoner laten zien.

De reden is eenvoudig: alle broeikasgassen die vrijkomen tijdens de teelt en verwerking worden in het producentenland geboekt, ook wanneer het voedsel naar het buitenland gaat. Een hoge uitstoot per inwoner weerspiegelt dus niet noodzakelijkerwijs wat de lokale bevolking eet, maar eerder hoeveel zij voor de wereldmarkt produceert. Met andere woorden, intensieve export verlegt de emissielast naar de exporteur, terwijl de consumptie elders plaatsvindt.

Dus hoe groter de landbouw­export per inwoner, des te hoger de landbouwgerelateerde CO₂-uitstoot per inwoner.



```python
import pandas as pd
import plotly.graph_objects as go
import plotly.io as pio
pio.renderers.default = "vscode"

# ───────────────────────────────────────────────────────────────
# 1. DATA INLEZEN
# ───────────────────────────────────────────────────────────────
emissions_animal = pd.read_csv("datasets/FAOSTAT_data_emissions_animal_6-23-2025.csv")
emissions_crops  = pd.read_csv("datasets/FAOSTAT_data_emissions_crops_6-23-2025-2.csv")
export_matrix    = pd.read_csv("datasets/matrix_export.csv")
population       = pd.read_csv("datasets/Population_E_All_Data_(Normalized).csv")

# ───────────────────────────────────────────────────────────────
# 2. GEGEVENS OPSCHONEN & SAMENVOEGEN
# ───────────────────────────────────────────────────────────────
population_clean = (
    population[
        (population["Element"] == "Total Population - Both sexes") &
        (population["Area"].isin([
            "Argentina", "Brazil", "India", "Thailand",
            "Netherlands", "United States of America"
        ]))
    ][["Area", "Year", "Value"]]
    .rename(columns={"Value": "Population (1000s)"})
)
population_clean["Year"] = population_clean["Year"].astype(int)

animal = emissions_animal[
    emissions_animal["Area"].isin([
        "Argentina", "Brazil", "India", "Thailand",
        "Netherlands", "United States of America"
    ]) &
    emissions_animal["Element"].str.contains("Emissions")
]

crops = emissions_crops[
    emissions_crops["Area"].isin([
        "Argentina", "Brazil", "India", "Thailand",
        "Netherlands", "United States of America"
    ]) &
    emissions_crops["Element"].str.contains("Emissions")
]

emissions_all = pd.concat([animal, crops])
emissions_all["Year"] = emissions_all["Year"].astype(int)
emissions_all = (
    emissions_all
    .groupby(["Area", "Year"])["Value"]
    .sum()
    .reset_index()
    .rename(columns={"Value": "Total Emissions (kt CO2eq)"})
)

export_volume = (
    export_matrix[
        (export_matrix["Element"] == "Export quantity") &
        (export_matrix["Reporter Countries"].isin([
            "Argentina", "Brazil", "India", "Thailand",
            "Netherlands", "United States of America"
        ]))
    ][["Reporter Countries", "Year", "Value"]]
    .groupby(["Reporter Countries", "Year"])
    .sum()
    .reset_index()
    .rename(columns={
        "Reporter Countries": "Area",
        "Value": "Total Export Volume (tonnes)"
    })
)
export_volume["Year"] = export_volume["Year"].astype(int)

df = (
    emissions_all
    .merge(population_clean, on=["Area", "Year"], how="left")
    .merge(export_volume,  on=["Area", "Year"], how="left")
)
df = df[df["Year"] >= 1985]

df["Emissions per Capita (tonnes CO2eq)"] = (
    df["Total Emissions (kt CO2eq)"] * 1_000
) / (df["Population (1000s)"] * 1_000)
df["Export per Capita (tonnes)"] = (
    df["Total Export Volume (tonnes)"]
) / (df["Population (1000s)"] * 1_000)
df["Emissions per Export Tonne (kg CO2eq/tonne)"] = (
    df["Total Emissions (kt CO2eq)"] * 1_000_000
) / df["Total Export Volume (tonnes)"]

metrics = {
    "Total Emissions (kt CO2eq)":               "Total Emissions (kt CO2eq)",
    "Emissions per Capita (tonnes CO2eq)":      "Emissions per Capita (tonnes CO2eq)",
    "Total Export Volume (tonnes)":             "Total Export Volume (tonnes)",
    "Export per Capita (tonnes)":               "Export per Capita (tonnes)",
    "Emissions per Export Tonne (kg CO2eq/tonne)":
        "Emissions per Export Tonne (kg CO2eq/tonne)"
}

# ───────────────────────────────────────────────────────────────
# 3. STIJLDEFINITIE – handmatig per land
# ───────────────────────────────────────────────────────────────
line_styles = {
    "Argentina": {
        "color": "#F79B9B",
        "dash":  "solid",
        "width": 3
    },
    "Brazil": {
        "color": "#F7CB9B",
        "dash":  "solid",
        "width": 3
    },
    "India": {
        "color": "#A1F79B",
        "dash":  "solid",
        "width": 3
    },
    "Thailand": {
        "color": "#6CBEF5",
        "dash":  "solid",
        "width": 3
    },
    "Netherlands": {
        "color": "#A39BF7",
        "dash":  "solid",
        "width": 3
    },
    "United States of America": {
        "color": "#F1A1F7",
        "dash":  "solid",
        "width": 3
    }
}

# ───────────────────────────────────────────────────────────────
# 4. FIGUUR MAKEN – met eigen stijlen
# ───────────────────────────────────────────────────────────────
fig = go.Figure()

for metric_idx, (metric_name, col) in enumerate(metrics.items()):
    for country in df["Area"].unique():
        subset  = df[df["Area"] == country]
        visible = (metric_idx == 0)

        style   = line_styles.get(country, {})
        fig.add_trace(
            go.Scatter(
                x=subset["Year"],
                y=subset[col],
                mode="lines",
                name=country,
                visible=visible,
                legendgroup=country,
                showlegend=True,
                line=dict(
                    color=style.get("color", None),
                    dash=style.get("dash",  "solid"),
                    width=style.get("width", 2)
                )
            )
        )

n_countries      = len(df["Area"].unique())
dropdown_buttons = []

for i, (metric_name, _) in enumerate(metrics.items()):
    visibility = [False] * len(metrics) * n_countries
    for j in range(n_countries):
        visibility[i * n_countries + j] = True
    dropdown_buttons.append(
        dict(
            label=metric_name,
            method="update",
            args=[
                {"visible": visibility},
                {"yaxis": {"title": metric_name}}
            ]
        )
    )

fig.update_layout(
    updatemenus=[{
        "buttons":       dropdown_buttons,
        "direction":     "down",
        "showactive":    True,
        "x":             1.03,
        "xanchor":       "left",
        "y":             0.95,
        "yanchor":       "top",
        "pad":           {"r": 0, "t": 0},
        "font":          {"size": 11},
        "bgcolor":       "white"
    }],
    title="Agricultural Emissions & Exports (1985–2020)",
    xaxis_title="Year",
    yaxis_title="Total Emissions (kt CO2eq)",
    legend_title="Country",
    legend=dict(
        x=1.03,
        xanchor="left",
        y=0.8,
        yanchor="top"
    ),
    margin=dict(r=160, l=80, t=80, b=60),
    height=600,
    width=900
)

# ───────────────────────────────────────────────────────────────
# 5. INTERFACE NAAR HET NEDERLANDS
# ───────────────────────────────────────────────────────────────

vertaling = {
    "Total Emissions (kt CO2eq)":               "Totale uitstoot (kiloton CO₂-equ.)",
    "Emissions per Capita (tonnes CO2eq)":      "Uitstoot per inwoner (ton CO₂-equ.)",
    "Total Export Volume (tonnes)":             "Totaal export­volume (ton)",
    "Export per Capita (tonnes)":               "Export per inwoner (ton)",
    "Emissions per Export Tonne (kg CO2eq/tonne)":
        "Uitstoot per ton export (kg CO₂-equ.)"
}

for btn in fig.layout.updatemenus[0].buttons:
    eng = btn.label
    if eng in vertaling:
        dut = vertaling[eng]
        btn.label                     = dut
        btn.args[1]["yaxis"]["title"] = dut

fig.update_layout(
    title="Landbouw­emissies en -export (1985–2020)",
    xaxis_title="Jaar",
    yaxis_title="Totale uitstoot (kt CO2-eq)",
    legend_title="Land"
)

# ───────────────────────────────────────────────────────────────
# 6. VISUALISATIE TONEN
# ───────────────────────────────────────────────────────────────
fig.show()

```



<span style="font-size:60%;">
kt = kilotonne&nbsp;(1 000 t)<br>
t = tonne&nbsp;(1 000 kg)<br>
CO₂-eq = CO₂-equivalent – alle broeikasgassen omgerekend naar de hoeveelheid CO₂ met hetzelfde opwarmend effect (GWP-100)<br>
Uitstoot per inwoner = totale uitstoot ÷ bevolking<br>
Uitstoot per ton export = totale uitstoot ÷ exportmassa
</span>

<span style="font-size: 60%;"> * Voor Argentinië ontbreken enkele gegevens voor 1999; daardoor verschijnt er een onderbreking in de lijn.</span>


##### Waarom deze landen?
We selecteerden Argentinië, Brazilië, Verenigde Staten, Thailand en India omdat zij samen alle logistieke archetypen laten zien die onze export-hypothese vereist. Argentinië en de VS hebben sinds de 19e eeuw directe Atlantische havens en bulkexport; Brazilië bezit kusthavens, maar zijn moderne sojazones liggen ruim duizend kilometer landinwaarts; Thailand is een smalle kuststaat met beperkte landbouwruimte; India heeft lange kusten, maar een moesson­gedreven binnenland­landbouw. Deze combinatie levert grote én kleine bevolkingen, uiteenlopende emissie­drivers (methaan uit vee, ontbossing, efficiënte graanlogistiek) en betrouwbare tijdreeksen sinds 1985. Andere landen zouden slechts variaties op dezelfde profielen toevoegen en het beeld vertroebelen. Met vijf cases tonen we dus alle relevante geografische combinaties — zonder redundantie — en testen we de rol van export in landbouw-CO₂ per inwoner op een compacte, maar volledige manier.

##### Analyse
Argentinië beschikt met de Río-de-la-Plata-delta over een korte Atlantische vaarroute. Die ligging maakte de Pampas al vroeg tot exportmotor. Vandaag springt Argentinië nog steeds uit met de hoogste export per inwoner én – in dezelfde grafiekset – de hoogste landbouw-CO₂ per inwoner. Open-luchtveeteelt op uitgestrekte graslanden produceert veel methaan, terwijl sojateelt grote oppervlakten beslaat. De historische exportpositie vertaalt zich hier rechtstreeks in een hoge emissiedruk per Argentijn.

Brazilië heeft oceaanhavens, maar de moderne soja- en veezones liggen ver landinwaarts in Cerrado en Amazone-overgangsgebieden. Daardoor blijft de export per inwoner bescheiden. Toch benadert de Braziliaanse uitstoot per inwoner de Argentijnse waarde. Geografisch verklaart ontbossing het verschil: de combinatie van savanne en regenwoud vergemakkelijkt het platbranden van nieuwe landbouwgrond, wat veel CO₂ oplevert, los van het exportvolume.

Verenigde Staten beschikken over zowel Atlantische en Pacifische havens als een fijnmazig netwerk van binnenwateren. Het absolute exportvolume is veruit het grootste, maar gedeeld door meer dan 330 miljoen inwoners blijft de export per hoofd slechts middelhoog. Bovendien is de emissie-intensiteit per ton laag dankzij zeer productieve Midwest-graanvelden dicht bij goedkope watertransporten. Daardoor daalt de uitstoot per Amerikaan tot ongeveer een derde van de Argentijnse waarde – een duidelijk breekpunt in de veronderstelde relatie.

Thailand is een smalle, kustgeoriënteerde staat met natuurlijke diepzee­havens in de Golf van Thailand. Toch blijft de export per inwoner laag: rijst is licht in gewicht en exportvolumes blijven bescheiden. De landbouw-CO₂ per inwoner volgt hetzelfde lage traject. Hier ondersteunt de geografie de hypothese: beperkte landbouwruimte en beperkte uitvoer leveren een lage emissiedruk op.

India bezit lange kusten, maar de bulk van de landbouw ligt op het ver landinwaarts gelegen Gangetische vlakkeland en bedient vooral de binnenlandse markt. De export per inwoner is vrijwel nihil en de uitstoot per inwoner het laagst, hoewel India in absolute termen de hoogste landbouwemissie heeft door de enorme bevolking. De logistieke drempel van moesson­wegen en lange koelketens beperkt de export ondanks de kustligging.

Een gunstige havenligging kan – zoals bij Argentinië – tot hoge export per inwoner én hoge CO₂ per inwoner leiden. Maar andere geografische factoren kunnen het patroon neutraliseren of omkeren. Grote bevolkingen en efficiënte binnenwaterlogistiek (VS) ‘verdunnen’ de emissies; interne afstanden tot havens en ontbossingsfronten (Brazilië, India) kunnen juist hoge uitstoot opleveren zonder uitzonderlijke export. De oorspronkelijke stelling is daarom slechts deels houdbaar: exportintensiteit is een belangrijke, maar geen voldoende, geografische verklaring voor verschillen in landbouw-CO₂ per hoofd.

### De geografische ligging van een land (en de bijbehorende klimaatzones) beïnvloedt welke gewassen en dieren men produceert — en dat heeft directe impact op de landbouwgerelateerde CO₂-uitstoot per capita.
Om zichtbaar te maken hoe klimaat en neerslag de landbouw-CO₂ bepalen, vergelijken we vier uitgesproken klimaatzones: het gematigde binnenland van de Verenigde Staten, het koele zeeklimaat van Nederland, het tropische savanne/moessonklimaat van Brazilië en het milde tropische hoogland van Kenia. In de volgende grafiekset ziet u per land het klimaat­patroon, het daaruit voortvloeiende gewasprofiel en de resulterende uitstoot per hoofd.


```python
import pandas as pd
import numpy as np
import plotly.graph_objects as go
from plotly.subplots import make_subplots
import plotly.io as pio
pio.renderers.default = "vscode"

# ==== MANUELE SPACING INSTELLEN ====
row_heights        = [0.25, 0.45, 0.30]
vertical_spacing   = 0.08
subplot_title_pad  = 20
top_margin         = 180
title_pad_top      = 30

# ==== COLOUR PANEL ====
colours = {
    'background':        '#FFFFFF',
    'font':              '#000000',
    'grid':              '#444444',
    'emissions_bar':     '#6094AA',
    'emissions_outline': '#000000',
    'production_bar':    "#65B1C5",
    'polar_bg':          '#FFFFFF',
    'precip_fill':       '#83CFE5',
    'precip_outline':    '#000000',
    'temp_line':         '#CA4A4A',
    'temp_marker':       '#CA4A4A'
}

# ==== NAME MAPPING ====
name_map = {
    'Netherlands (Kingdom of the)': 'Netherlands',
    'United States of America':      'United States'
}

# ==== DUTCH TRANSLATIONS ====
dutch_country_names = {
    'Brazil': 'Brazilië',
    'Netherlands': 'Nederland',
    'United States': 'Verenigde Staten',
    'Kenya': 'Kenia'
}

dutch_month_names = {
    'Jan': 'Jan','Feb': 'Feb','Mar': 'Maa','Apr': 'Apr',
    'May': 'Mei','Jun': 'Jun','Jul': 'Jul','Aug': 'Aug',
    'Sep': 'Sep','Oct': 'Okt','Nov': 'Nov','Dec': 'Dec'
}

dutch_category_names = {
    'Fruits': 'Fruit',
    'Vegetables': 'Groenten',
    'Grains & Cereals': 'Granen en Ceralia',
    'Oil Crops': 'Oliegewassen',
    'Livestock & Animal Products': 'Vee en Dierlijke Producten',
    'Sugar cane': 'Suikerriet',
    'Cotton': 'Katoen',
    'Rubber': 'Rubber',
    'Silk': 'Zijde',
    'Jute/Hemp': 'Jute/Hennep'
}

# ==== CATEGORY MAP ====
category_map = {
    'Apples':'Fruits','Bananas':'Fruits','Oranges':'Fruits','Pineapples':'Fruits',
    'Mangoes, mangosteens, guavas':'Fruits','Papayas':'Fruits','Watermelons':'Fruits',
    'Grapes':'Fruits','Lemons and limes':'Fruits','Tangerines, mandarins, clementines':'Fruits',
    'Cantaloupes and other melons':'Fruits','Pears':'Fruits','Peaches and nectarines':'Fruits',
    'Avocados':'Fruits','Strawberries':'Fruits','Plums and sloes':'Fruits','Cherries':'Fruits',
    'Pomegranates':'Fruits','Pomelos and grapefruits':'Fruits','Figs':'Fruits','Persimmons':'Fruits',
    'Currants':'Fruits','Sour cherries':'Fruits','Olives':'Fruits',
    'Plantains and cooking bananas':'Fruits','Cranberries':'Fruits',
    'Tomatoes':'Vegetables','Onions, dry':'Vegetables','Cabbages':'Vegetables',
    'Carrots and turnips':'Vegetables','Lettuce and chicory':'Vegetables','Spinach':'Vegetables',
    'Cauliflowers and broccoli':'Vegetables','Cucumbers and gherkins':'Vegetables',
    'Pumpkins, squash and gourds':'Vegetables','Garlic':'Vegetables',
    'Leeks, other alliaceous vegetables':'Vegetables','Mushrooms and truffles':'Vegetables',
    'Eggplants (aubergines)':'Vegetables','Peas, green':'Vegetables',
    'Artichokes':'Vegetables','Asparagus':'Vegetables',
    'Wheat':'Grains & Cereals','Maize':'Grains & Cereals','Rice, paddy':'Grains & Cereals',
    'Barley':'Grains & Cereals','Sorghum':'Grains & Cereals','Millet':'Grains & Cereals',
    'Oats':'Grains & Cereals','Rye':'Grains & Cereals','Triticale':'Grains & Cereals',
    'Quinoa':'Grains & Cereals','Canary seed':'Grains & Cereals','Cereals n.e.c.':'Grains & Cereals',
    'Soybeans':'Oil Crops','Rapeseed':'Oil Crops','Sunflower seed':'Oil Crops','Oil palm fruit':'Oil Crops',
    'Groundnuts, in shell':'Oil Crops','Sesame seed':'Oil Crops','Safflower seed':'Oil Crops',
    'Mustard seed':'Oil Crops','Flaxseed':'Oil Crops','Hempseed':'Oil Crops','Linseed':'Oil Crops',
    'Palm kernels':'Oil Crops','Castor oil seeds':'Oil Crops',
    'Cattle':'Livestock & Animal Products','Buffalo':'Livestock & Animal Products','Sheep':'Livestock & Animal Products',
    'Goats':'Livestock & Animal Products','Chickens':'Livestock & Animal Products','Ducks':'Livestock & Animal Products',
    'Turkeys':'Livestock & Animal Products','Geese and guinea fowls':'Livestock & Animal Products',
    'Pigeons, other birds':'Livestock & Animal Products','Rabbits and hares':'Livestock & Animal Products',
    'Camels':'Livestock & Animal Products','Horses':'Livestock & Animal Products','Mules':'Livestock & Animal Products',
    'Hen eggs, in shell':'Livestock & Animal Products','Raw milk of cattle':'Livestock & Animal Products',
    'Raw milk of buffalo':'Livestock & Animal Products','Raw milk of sheep':'Livestock & Animal Products',
    'Raw milk of goats':'Livestock & Animal Products','Honey':'Livestock & Animal Products',
    'Livestock fat':'Livestock & Animal Products','Meat':'Livestock & Animal Products',
    'Sugar cane':'Sugar cane',
    'Cotton lint':'Cotton','Cottonseed':'Cotton',
    'Rubber, natural':'Rubber','Natural rubber in primary forms':'Rubber',
    'Silk-worm cocoons, reelable':'Silk','Raw silk (not thrown)':'Silk',
    'Jute, raw or retted':'Jute/Hemp','True hemp, raw or retted':'Jute/Hemp',
    'Ramie, raw or retted':'Jute/Hemp','Kenaf, and other textile bast fibres, raw or retted':'Jute/Hemp',
    'Sisal, raw':'Jute/Hemp','Coir, raw':'Jute/Hemp','Kapok fibre, raw':'Jute/Hemp'
}

# ==== 1) LOAD & PREP ====
prod = pd.read_csv('datasets/FAOSTAT_production_arg3.csv')
prod['Country'] = prod['Area'].replace(name_map)
prod['Category'] = prod['Item'].map(category_map)
prod_df = prod.query("Element=='Production' & Year==2022 & Unit=='t'").dropna(subset=['Category'])
grouped_prod = prod_df.groupby(['Country','Category'])['Value'].sum().reset_index()

# ==== 2) EMISSIONS ====
em  = pd.read_csv('datasets/FAOSTAT_data_en_6-26-2025-6.csv')
pop = pd.read_csv('datasets/API_SP.POP.TOTL_DS2_en_csv_v2_124794.csv', skiprows=4)
em['Area'] = em['Area'].replace(name_map)
pop['Country Name'] = pop['Country Name'].replace(name_map)

countries = ['Brazil','Netherlands','United States','Kenya']
em2    = em.query("Year==2022 and Area in @countries")
em_tot = em2.groupby('Area')['Value'].sum().reset_index().rename(columns={'Area':'Country','Value':'Total_Emissions'})
pop2   = pop[['Country Name','2022']].rename(columns={'Country Name':'Country','2022':'Population'})
merged_em = pd.merge(em_tot, pop2, on='Country')
merged_em['Emissions_per_capita'] = merged_em['Total_Emissions'] / merged_em['Population']

# ==== 3) TEMP & PRECIP ====
tp = pd.read_csv('datasets/combined_2022.csv')
tp['name'] = tp['name'].replace({'United States of America':'United States'})
tp = tp[tp['name'].isin(countries)]
tp['Month']     = pd.to_datetime(tp['Month'], format='%Y-%m')
tp['Month_str'] = tp['Month'].dt.strftime('%b').map(dutch_month_names)
month_order     = tp.sort_values('Month')['Month_str'].unique()
y_temp_min, y_temp_max = tp['Temperature'].min(), tp['Temperature'].max()

# ==== GRAPH TITLES ====
graph_titles = [
    "CO₂-uitstoot per hoofd van de bevolking (2022)",
    "Landbouwproductie per categorie (2022)",
    "Maandelijkse temperatuur en neerslag (2022)"
]

# ==== BUILD FIGURE ====
fig = make_subplots(
    rows=3, cols=1,
    specs=[[{'type':'xy'}],[{'type':'polar'}],[{'type':'xy','secondary_y':True}]],
    row_heights=row_heights,
    vertical_spacing=vertical_spacing,
    subplot_titles=("Grafiek 1","Grafiek 2","Grafiek 3")
)

categories       = sorted(grouped_prod['Category'].unique())
dutch_categories = [dutch_category_names.get(cat,cat) for cat in categories]
angles           = np.linspace(0, 360, len(categories), endpoint=False)

for i, country in enumerate(countries):
    val = merged_em.loc[merged_em['Country']==country, 'Emissions_per_capita'].iat[0]
    fig.add_trace(go.Bar(
        x=[dutch_country_names[country]], y=[val],
        marker=dict(color=colours['emissions_bar'],
                    line=dict(color=colours['emissions_outline'], width=2)),
        visible=(i==0)
    ), row=1, col=1)

    dfp = (grouped_prod.query("Country==@country")
           .set_index('Category').reindex(categories).reset_index())
    fig.add_trace(go.Barpolar(
        r=dfp['Value'],
        theta=[angles[categories.index(c)] for c in dfp['Category']],
        width=[360/len(categories)*0.8]*len(categories),
        marker_color=colours['production_bar'],
        visible=(i==0)
    ), row=2, col=1)

    dft = tp[tp['name']==country]
    fig.add_trace(go.Bar(
        x=dft['Month_str'], y=dft['Precipitation'],
        marker=dict(color=colours['precip_fill'],
                    line=dict(color=colours['precip_outline'], width=1)),
        opacity=0.7, visible=(i==0)
    ), row=3, col=1, secondary_y=True)

    fig.add_trace(go.Scatter(
        x=dft['Month_str'], y=dft['Temperature'],
        mode='lines+markers',
        line=dict(color=colours['temp_line']),
        marker=dict(color=colours['temp_marker']),
        visible=(i==0)
    ), row=3, col=1, secondary_y=False)

buttons = []
for idx, country in enumerate(countries):
    vis = [False]*len(fig.data)
    for j in range(4):
        vis[4*idx + j] = True
    buttons.append(dict(
        label=dutch_country_names[country],
        method='update',
        args=[{'visible': vis}, {}]
    ))

fig.update_layout(
    updatemenus=[dict(
        active=0,
        buttons=buttons,
        x=0.1, y=1.38, xanchor='left', yanchor='top',
        direction='down',
        pad=dict(l=10, r=10, t=10, b=10),   # <-- hier de padding instellen
        font=dict(size=16)                   # <-- hier de lettergrootte (en dus dropdown-grootte)
    )],
    margin=dict(l=50, r=50, t=top_margin, b=50),
    plot_bgcolor=colours['background'],
    paper_bgcolor=colours['background'],
    font=dict(color=colours['font']),
    showlegend=False,
    width=700,
    height=1350,
    polar=dict(bgcolor=colours['polar_bg'])
)

fig.add_annotation(
    text=(
        "Alle indicatoren per land<br>"
        "Grafiek 1: CO₂-uitstoot per hoofd van de bevolking (2022)<br>"
        "Grafiek 2: Landbouwproductie per categorie (2022)<br>"
        "Grafiek 3: Maandelijkse temperatuur en neerslag (2022)"
    ),
    xref="paper", yref="paper",
    x=0.5, y=1.2,
    showarrow=False,
    font=dict(size=16),
    align="center"
)

for ann in fig.layout.annotations:
    ann.yshift = -subplot_title_pad

fig.update_yaxes(row=1, col=1,
    range=[0,2], dtick=0.5,
    showgrid=True, gridcolor=colours['grid']
)
fig.update_layout(polar=dict(
    radialaxis=dict(showgrid=True, gridcolor=colours['grid']),
    angularaxis=dict(showgrid=True, gridcolor=colours['grid'],
                     tickvals=angles, ticktext=dutch_categories)
))
fig.update_xaxes(row=3, col=1,
    type='category',
    categoryorder='array',
    categoryarray=month_order,
    showgrid=True, gridcolor=colours['grid']
)
fig.update_yaxes(row=3, col=1, secondary_y=False,
    range=[y_temp_min,y_temp_max], dtick=5,
    showgrid=True, gridcolor=colours['grid']
)
fig.update_yaxes(row=3, col=1, secondary_y=True, showgrid=False)

fig.show()

```



<span style="font-size: 60%;"> * We hebben de volumes van snijbloemen niet in de dataset opgenomen, omdat voor sommige landen (bv. Nederland en Kenia) betrouwbare productie­cijfers beschikbaar zijn, terwijl voor andere landen alleen exportwaarden of grove schattingen bestaan. Door deze ongelijksoortige bronnen zouden we een scheef beeld creëren. We vonden het daarom beter om de rol van sierteelt kwalitatief in de analyse te bespreken in plaats van kwantitatief in de grafiek te tonen; zo blijft de vergelijking tussen landen consistent en evenwichtig. Bovendien is snijbloementeelt in onze landenselectie feitelijk alleen relevant voor Nederland en Kenia; bij de Verenigde Staten en Brazilië speelt zij geen noemenswaardige rol.</br> * Let op: de schaalverdeling van de assen wisselt per land bij Grafiek 2.
Dit is bewust zo gekozen om zowel hoge als lage waarden goed te laten uitkomen. Bij een uniforme asschaal zouden extreme uitschieters de weergave van kleinere waarden grotendeels verbergen. </span>


##### Waarom deze landen?
We selecteerden Verenigde Staten, Nederland, Brazilië en Kenia omdat zij samen alle relevante landbouwklimaten afdekken. De VS vertegenwoordigt het gematigde landklimaat, Nederland het koele zeeklimaat, Brazilië het tropische savanne/moessonklimaat en Kenia het milde tropische hoogland.

##### Analyse 
De Verenigde Staten liggen in een gematigde landklimaatzone (Köppen Dfa/Cfa) met koude winters, warme zomers en de meeste neerslag in de groeimaanden. De temperatuur- en neerslaggrafiek laat duidelijk zien dat slechts één lang, warm seizoen geschikt is voor akkerbouw. In dat venster floreren granen als maïs, tarwe en soja, en uitgestrekte graslanden bieden volop weide voor rundvee. Die gewassen domineren het radardiagram; tropische opties, zoals suikerriet, ontbreken omdat de winters te koud en de zomers niet vochtig genoeg zijn. Runderen produceren veel methaan en vragen extra voergranen, terwijl kunstmest bij graanteelt tot extra lachgas leidt. Dit emissie-intensieve pakket verklaart de hoge landbouw-CO₂-uitstoot per inwoner die in de eerste grafiek zo nadrukkelijk boven de andere landen uitsteekt. 

Het koele, natte zeeklimaat van Nederland (Köppen Cfb) kent milde zomers en winters rond drie graden. Gras groeit daardoor vrijwel het hele jaar, wat een grote melkveesector ondersteunt; dat segment is zichtbaar in het radardiagram. Daarnaast is Nederland, ondanks zijn geringe oppervlakte, ‘s werelds grootste exporteur van snijbloemen. Omdat het winterhalfjaar te koud en te donker is, worden tulpen, rozen en gerbera’s grotendeels in verwarmde en belichte kassen geteeld. Die kassen compenseren het klimaat, maar vergen veel gas en stroom. Zo stuwt juist de sierteelt de landbouw-CO₂-uitstoot per inwoner omhoog, al is de geproduceerde massa relatief klein. De balkgrafiek weerspiegelt dit: een compact land met een opvallend hoge uitstoot, puur doordat het klimaat energie-intensieve kasbloemen afdwingt.

In Kenia toont de klimaatgrafiek jaarrond milde twintig graden met twee regenpieken. Dit hooglandklimaat (Köppen Cwb) is uitgelezen voor open-luchtteelt van thee, groenten én snijbloemen. Kenia is na Nederland de grootste bloemenexporteur ter wereld, maar hoeft voor rozen en alstroemeria nauwelijks kunstmatige verwarming of verlichting te gebruiken. De radardiagram laat die volumes niet afzonderlijk zien, toch komen ze uit velden rond Naivasha waar natuurlijke daglichturen en gelijkmatige temperaturen volstaan. Zonder kassen en met een bescheiden veestapel blijft Kenia’s landbouw-CO₂-uitstoot per inwoner vrijwel nihil, zoals de balkgrafiek bevestigt. Zo illustreert hetzelfde product – bloemen – hoe een gunstig klimaat de emissievoetafdruk drastisch kan verlagen.

Brazilië kent een afwisseling van tropisch moesson- en savanneklimaat (Köppen Am/Aw) met een uitgesproken nat seizoen van november tot april en een drogere winter. Tijdens de vochtige, warme maanden groeien soja en suikerriet uitzonderlijk goed; in sommige regio’s is zelfs een tweede, korter maïsgewas na de soja mogelijk. Runderen kunnen jaar rond in open weiden grazen, omdat vorst ontbreekt. Deze patronen komen terug in het radardiagram, waar suikerriet, soja en vee domineren. Omdat verwarming, kunstlicht of winterstallen overbodig zijn, blijft de CO₂-uitstoot per inwoner matig. Toch leidt het drogere winterseizoen indirect tot extra emissies: savanne en bos worden vaak afgebrand om nieuwe landbouwgrond te winnen. Zo koppelt het klimaat zowel de productieve kansen als de emissierisico’s van Brazilië. Ondanks een landoppervlak vergelijkbaar met de VS laat Brazilië zien dat een warm, vochtig klimaat de per-capita uitstoot kan drukken wanneer kassen en wintervoer overbodig zijn.

