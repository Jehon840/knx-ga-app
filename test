
import streamlit as st
import xml.etree.ElementTree as ET
import io
import math

st.set_page_config(page_title="KNX ETS Generator – Auto-MG + Namen", layout="wide")
st.title("KNX ETS XML Generator – automatische Mittelgruppen mit Namen")

anzahl_hauptgruppen = st.number_input("Anzahl Hauptgruppen", min_value=1, max_value=32, step=1)
hauptgruppen = []
for i in range(anzahl_hauptgruppen):
    with st.expander(f"Hauptgruppe {i+1}"):
        nummer = st.selectbox(f"Hauptgruppe {i+1} Nummer (0–31)", list(range(32)), key=f"hgnum_{i}")
        name = st.text_input(f"Name Hauptgruppe {nummer}", f"{i+1}.OG", key=f"hgname_{i}")
        raumanzahl = st.number_input(f"Anzahl Räume in Hauptgruppe {nummer}", 1, 10, key=f"raumanza_{i}")
        raeume = []
        for r in range(raumanzahl):
            raumname = st.text_input(f"Raum {r+1} Name", f"{nummer:02d}.{r+1}", key=f"hg{i}_raum{r}")
            raeume.append(raumname)

        funktionen = {}
        for raum in raeume:
            gewerke = st.multiselect(f"Gewerke in {raum}", ["Licht 5er", "Licht 10er", "Storen", "Heizung", "Andere"], key=f"g_{i}_{raum}")
            funktionen[raum] = {}
            for g in gewerke:
                anzahl = st.selectbox(f"{g}-Gruppen in {raum}", list(range(1, 21)), key=f"{raum}_{i}_{g}")
                funktionen[raum][g] = anzahl
        hauptgruppen.append({"nummer": nummer, "name": name, "funktionen": funktionen})

funktionen_typ = {
    "Licht 5er": ["ea", "dimm", "wert", "rm 1bit", "rm wert"],
    "Licht 10er": ["ea", "dimm", "wert", "rm 1bit", "rm wert", "Reserve6", "Reserve7", "Reserve8", "Reserve9", "Reserve10"],
    "Storen": ["AUF/AB", "STOPP/LAMELLEN", "POSITION HÖHE", "POSITION LAMELLEN", "BESCHATTUNG", "SPERREN", "STATUS POSITION HÖHE", "STATUS POSITION LAMELLEN", "Reserve9", "Reserve10"],
    "Heizung": ["STELLGRÖSSE", "IST", "BASIS-SOLL", "RM AKTUELLER SOLLWERT", "UMSCHALTEN BETRIEBSART", "STATUS BETRIEBSART", "STÖRUNG", "SPERREN", "Reserve9", "Reserve10"],
    "Andere": ["ea", "status", "wert", "toggle", "reserve"]
}

def ga_address(haupt, mittel, sub):
    return f"{haupt}/{mittel}/{sub}"

def ga_to_id(haupt, mittel, sub):
    return (haupt << 11) + (mittel << 8) + sub

def generate_structure(hauptgruppen):
    struktur = []
    for hg in hauptgruppen:
        hnum = hg["nummer"]
        funktionen = hg["funktionen"]

        grouped = {}  # key = mittelgruppe nummer, value = list of GA
        names = {}    # key = mittelgruppe nummer, value = name
        mg_index = 1  # MG0 = Zentral
        mg_tracker = {}  # GA Index Tracker per MG
        grouped[0] = []
        names[0] = "Zentral"

        used_mg = {}

        # Gewerke zählen und Mittelgruppen reservieren
        for gewerk, funk_typ in funktionen_typ.items():
            ga_per_group = len(funk_typ)
            total_groups = 0
            for raum, gewerke in funktionen.items():
                if gewerk in gewerke:
                    total_groups += gewerke[gewerk]
            total_ga = total_groups * ga_per_group
            needed_mg = math.ceil(total_ga / 256)
            for mg in range(needed_mg):
                grouped[mg_index + mg] = []
                mg_tracker[(gewerk, mg)] = 0
                if needed_mg == 1:
                    names[mg_index + mg] = gewerk
                else:
                    names[mg_index + mg] = f"{gewerk} ({mg+1})"
            used_mg[gewerk] = (mg_index, needed_mg)
            mg_index += needed_mg

        # GA verteilen
        for gewerk, funk_typ in funktionen_typ.items():
            ga_per_group = len(funk_typ)
            current_mg_idx = used_mg[gewerk][0]
            ga_counter = 0
            for raum, gewerke in funktionen.items():
                if gewerk not in gewerke:
                    continue
                for g in range(gewerke[gewerk]):
                    prefix = f"{raum}.{g+1}"
                    for f in funk_typ:
                        mg_slot = current_mg_idx + (ga_counter // 256)
                        subindex = mg_tracker[(gewerk, mg_slot - current_mg_idx)]
                        address = ga_address(hnum, mg_slot, subindex)
                        grouped[mg_slot].append({
                            "name": f"{prefix} {f}" if f else prefix,
                            "address": address,
                            "ga_id": ga_to_id(hnum, mg_slot, subindex)
                        })
                        mg_tracker[(gewerk, mg_slot - current_mg_idx)] += 1
                        ga_counter += 1

        struktur.append({"name": hg["name"], "nummer": hnum, "gruppen": grouped, "mg_names": names})
    return struktur

def export_full_ets_structure(struktur):
    ns = "http://knx.org/xml/ga-export/01"
    ET.register_namespace("", ns)
    root = ET.Element(f"{{{ns}}}GroupAddress-Export")

    for hg in struktur:
        hnum = hg["nummer"]
        h_start = str(hnum * 2048)
        h_end = str((hnum + 1) * 2048 - 1)
        h_grp = ET.SubElement(root, "GroupRange", Name=hg["name"], RangeStart=h_start, RangeEnd=h_end)

        for mg, eintraege in sorted(hg["gruppen"].items()):
            if not eintraege:
                continue
            mg_start = str(hnum * 2048 + mg * 256)
            mg_end = str(hnum * 2048 + (mg + 1) * 256 - 1)
            name = hg["mg_names"].get(mg, f"Mittelgruppe {mg}")
            mg_elem = ET.SubElement(h_grp, "GroupRange", Name=name, RangeStart=mg_start, RangeEnd=mg_end)
            for ga in eintraege:
                ET.SubElement(mg_elem, "GroupAddress", Name=ga["name"], Address=ga["address"])

    return ET.ElementTree(root)

if st.button("XML generieren"):
    struktur = generate_structure(hauptgruppen)
    tree = export_full_ets_structure(struktur)
    buffer = io.BytesIO()
    tree.write(buffer, encoding="utf-8", xml_declaration=True)
    st.download_button("ETS XML herunterladen", buffer.getvalue(), "knx_ets_named_mg.xml", "application/xml")
