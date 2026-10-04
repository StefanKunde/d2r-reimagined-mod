# MAP_FEASIBILITY – Neue Gebiete (Maps/Areas) für D2R Reimagined

Phase 1 (nur Analyse) + **Phase 1b „Fahrender Händler“ (Abschnitt 11)**. Stand: 2026-10-04. Phase 2 (Gebiet) ist **pausiert**, bis die Händler-Frage geklärt ist. Basis: Branch `next` @ `bba512f` (identisch mit `origin/next`),
SDK-Klon `../PluginSDK` @ `717f727` (SDK 0.3.0, Plugin-ABI 4). Es wurde **keine bestehende Datei geändert**
(einzige Nebenwirkung: die flache Git-Historie wurde per `git fetch --unshallow` vervollständigt – nur `.git`,
2020 Commits; sie liefert die wichtigsten Belege dieses Berichts).

**Legende**
- **[BELEGT]** – in Repo, SDK oder Git-Historie nachprüfbar (Pfad:Zeile bzw. Commit-Hash angegeben).
- **[ANNAHME]** – Vermutung oder Allgemeinwissen über D2/D2R, *nicht* aus den Dateien belegt.
- Zeilennummern beziehen sich auf den aktuellen Stand von `next` (Zeile 1 = Header).

**Grenzen dieser Analyse:** Die Sandbox ist Linux ohne Diablo II: Resurrected und ohne Wine. Der Linter
(`lint/d2rlint.exe`, PE32+ Windows-Binary) konnte **nicht** ausgeführt werden; nichts davon wurde im Spiel getestet.
Alles, was „funktioniert“ heißt, heißt: *im Repo existiert ein früher ausgelieferter Präzedenzfall*, nicht: *von mir getestet*.

---

## 0. Kurzfazit

**Eingeschränkt machbar – und zwar gut machbar auf der Datenebene, nicht machbar per Plugin.**

| Frage | Urteil | Begründung |
|---|---|---|
| Neues Gebiet **ohne neues Layout** (vorhandene DS1/Tiles wiederverwenden, Maze-Level) | **machbar** | Rein tabellengetrieben. Reimagined hat das bereits mehrfach getan: Pitzilla-Höhle (3 Level, Ids 137–139, 2024/25), Forsaken Labyrinth + „Panda“-Portal-Level (Ids 137–155, Commit `7f0ed3b3`, 2025-08-03), Pandemonium-Finale (Id 136, nutzt `Tri_Town4.ds1` von Tristram, `lvlprest.txt:1093`). |
| Neues Gebiet **mit eigenem Layout** (neue/umgebaute `.ds1`) | **eingeschränkt machbar** | Geht (Präzedenz: `labyrinth00/05/10.ds1`, `pzboss1/pzend.ds1`), braucht aber **externe Binär-Editoren** (DS1) und für HD je **ein HD-Preset-JSON** (0,2–5 MB, `data/hd/env/preset/…`). Nicht aus dem Repo heraus mit Textwerkzeugen erzeugbar; hier nicht testbar. |
| Neues Gebiet **per D2RLoader-Plugin** erzeugen/ändern | **nicht machbar (mit der öffentlichen API)** | Das SDK bietet **keinen** Zugriff auf Level-Generierung, schreibt keine Stock-Tabellen und kann keine DS1/DT1 registrieren. Einzig mögliche Route wären rohe Inline-Hooks auf build-spezifische RVAs – dafür liefert das SDK weder Offsets noch Symbole. Ein Plugin kann Level-Tabellen nur **lesen** und auf Level-Wechsel **reagieren** (Abschnitt 6). |
| Dauerhafte Wartbarkeit | **Risiko mittel–hoch** | Level-Ids 0–137 sind lückenlos belegt; **Blizzard selbst hat 137 mit dem RotW-Update (Colossal Summit) vergeben** und damit Reimagined-Level verdrängt (Abschnitt 8, R1). Jede neue Id ≥ 138 kann in einem künftigen Blizzard- oder Reimagined-Update kollidieren. |

**Empfehlung:** Plugin-Teil vorerst weglassen. Phase 2 als rein datengetriebenen Prototyp bauen (Abschnitt 9):
ein kleines Gebiet mit Id 138, Zugang über ein Cube-Portal (wie Colossal Summit / Pitzilla), ohne Änderung bestehender Zeilen.

---

## 1. Überblick: Repo, Build, Installation

[BELEGT]
- `README.md`: Quellcode der Mod „Reimagined“; PRs gegen `next`; Launcher: <https://github.com/D2R-Reimagined/reimagined-launcher>.
- `modinfo.json`: `{ "name": "Reimagined", "version": "3.0.12", "savepath": "ReimaginedThree/" }`.
- `.editorconfig`: 4 Leerzeichen. `.gitignore`: `*.png`, `*.bak`, `build/*`, `.studio/*`, IDE-Dateien.
- Es gibt **keinen Build-Schritt**: `data/` wird 1:1 als Mod-Ordner kopiert. `scripts/copy-files.bat`:
  `robocopy "C:\dev\d2r\d2r-reimagined\data" "C:\Program Files (x86)\Diablo II Resurrected\mods\Reimagined\Reimagined.mpq\data" /MIR /E /IS /IT`
  (Zielordner heißt `Reimagined.mpq` – ein Ordner, kein gepacktes MPQ; Pfade hartcodiert; `/MIR` löscht im Ziel überzählige Dateien).
- `D2R Data Guide.url` → <https://eezstreet.github.io/d2rdoc/guides/getting-started.html>.
- `data/global/dataversionbuild.txt` = `93847` (Daten-Build-Nummer der Basisdaten).
- Verzeichnisstruktur (`data/`): `global/` (excel, tiles, ui, items, `*.d2`), `hd/` (env, vfx, character, items, global/excel-JSON …), `local/lng/strings/` (8 String-JSONs).
- 2531 Dateien: 666 `.sprite`, 523 `.flac`, 446 `.json`, 461 `.dc6/.DC6`, 111 `.txt`, **82 `.ds1`**, 73 `.timelines`, 59 `.texture`, 49 `.particles`, 35 `.model`, 5 leere `combined_ds1.bin`. **Keine einzige `.dt1`.**

**Zwei Excel-Bänke – wichtig für jede Änderung** [BELEGT, Mechanismus teils ANNAHME]
- `data/global/excel/*.txt` (57 Dateien) und `data/global/excel/base/*.txt` (55 Dateien) sind **inhaltlich identisch** (geprüft: alle Dateien gleich bis auf `treasureclassex.txt`; Unterschied bei `levels.txt` nur Zeilenenden: `excel/` = CRLF, `excel/base/` = LF).
- `excel/a_copy_excel_base.bat` kopiert `excel/*.txt` → `excel/base/`, **ausgenommen `treasureclassex.txt`**.
- Das SDK benennt die Bänke: *RotW:* `data/global/excel/…`, *Base:* `data/global/excel/base/…` (`PluginSDK/README.md:662-663`, `include/D2RLPlugin/custom_tables.h:81-83`; Enum `Bank { Classic, Lod, Rotw }`, `data_tables.h:29-33`).
  → `excel/` = Reign-of-the-Warlock-Bank, `excel/base/` = Classic/LoD-Bank. **[ANNAHME]**, dass das für die Stock-Tabellen identisch gilt (starkes Indiz: `base/` entstand im RotW-Import-Commit `11aa02dc`, 2026-02-15).
- Konsequenz: **Jede Änderung an einer Excel-Tabelle muss in beiden Ordnern erfolgen** (Reimagined-Commits ändern stets beide, z. B. `852c28a1`, `5ead2711`).

---

## 2. Relevante Dateien (Pfad · Größe · Zweck)

Größen in Bytes (`data/global/excel/…`; `excel/base/…` je nahezu gleich groß).

### 2.1 Im Repo vorhanden

| Pfad | Größe | Zeilen/Einträge | Zweck bei Gebieten |
|---|---:|---|---|
| `data/global/excel/levels.txt` | 81 503 | 140 Z. (139 Zeilen, 188 Spalten) | **Master-Tabelle aller Gebiete** (Id, Act, Typ, Größe, Monster, Vis/Warp, Strings, Waypoint, LevelGroup) |
| `data/global/excel/lvlprest.txt` | 129 582 | 1094 Z. (1093 Rows) | **Preset-Level**: welche `.ds1` ein Level (LevelId) bzw. ein Outdoor-Baustein nutzt, `Dt1Mask`, Größe, Populate |
| `data/global/excel/lvlmaze.txt` | 3 212 | 83 Z. (82 Rows) | **Maze-Level** (zufällig generierte Dungeons): Raumzahl je Schwierigkeit, Größe, Merge |
| `data/global/excel/lvlwarp.txt` | 5 515 | 91 Z. (90 Rows) | Treppen/Warp-Kacheln (Klick-/Laufbereich, Richtung, UniqueId) – Ziel der `Warp0–7`-Spalten |
| `data/global/excel/levelgroups.txt` | 5 837 | 102 Z. (101 Rows) | Gebietsgruppen (`LevelGroup`-Spalte in `levels.txt`; Terror-Zonen/Completion) |
| `data/global/excel/actinfo.txt` | 1 488 | 6 Z. | Pro Act: Stadt, bis zu **9 Waypoints** (`waypoint1–9`), NPC-/Wandering-Parameter |
| `data/global/excel/automap.txt` | 153 739 | 3288 Z. | Automap-Kacheln, **nach Level-Typ** (Spalte 1 = Typname, z. B. `5 Barricade`), nicht nach Level-Id |
| `data/global/excel/objects.txt` | 208 570 | 588 Rows | Objekte (Altäre, Statuen, Truhen …), von DS1-Layouts referenziert |
| `data/global/excel/monstats.txt` / `monstats2.txt` | 540 378 / 144 784 | 774 / – Rows | Monster-Definitionen (Level-Monsterlisten `mon1…` verweisen auf `monstats.Id`) |
| `data/global/excel/monpreset.txt` | 4 419 | 241 Z. | Zuordnung DS1-Platzhalter (`Act`,`Place`,`DS1 ID#`) → Monster/NPC |
| `data/global/excel/superuniques.txt` | 12 706 | – | Bossmonster (Super Uniques) |
| `data/global/excel/cubemain.txt` | 3 664 460 | 15 901 Z. | Horadrim-Würfel; **Portal-Rezepte** („Red Portal“) |
| `data/global/excel/misc.txt`, `treasureclassex.txt` | 111 273 / 193 812 | – | Portal-Materialien (`ua1–ua5`) bzw. deren Drops |
| `data/local/lng/strings/levels.json` | 268 578 | 422 Einträge × 13 Sprachen | **Gebietsnamen** (Keys = `LevelName`/`LevelWarp`/`LevelEntry` aus `levels.txt`) |
| `data/local/lng/strings/monsters.json`, `ui.json`, `item-*.json`, `skills.json` | 0,13–1,8 MB | – | übrige Strings (je 13 Sprachen) |
| `data/global/tiles/**.ds1` | 2 KB – 147 KB | **82 Dateien** | Layout-Dateien (Wände/Böden/Objekte/Tags); nur die von Reimagined **geänderten/neuen**; z. B. `act1/outdoors/bivouac.ds1` 5 305, `labyrinth/labyrinth00.ds1` 31 578, `expansion/town/townwest.ds1` 76 733, `act2/town/lutn.ds1` 146 900 |
| `data/global/tiles/*/combined_ds1.bin` | **0** (5 Stück) | leer | Zweck nicht dokumentiert (eingeführt in `ee585c65`, 2024-04-27) **[ANNAHME]:** Platzhalter, damit lose DS1 statt des vorgepackten Containers geladen werden |
| `data/hd/env/preset/**.json` | 218 KB – 5,1 MB | **63 Dateien** | **HD-3D-Szenen** je DS1 (Entities, Modelle, Texturen, Physik, `biomeFilename`), z. B. `act1/outdoors/bivouac.json` 617 745, `labyrinth/labyrinth00.json` 2 761 027, `expansion/town/townwest.json` 5 105 608 |
| `data/hd/global/excel/desecratedzones.json` | 148 110 | 2592 Z. | **Terror-Zonen**-Konfiguration: Liste von `level_id` je Zone (max. `level_id` = 132; 137 nicht enthalten) |
| `data/hd/env/weather/vis_event_map.json` | 4 373 | – | Wetter-/Vis-Events |
| `data/global/ui/layouts/waypointspanelexpansionhd.json` | 1 619 | 52 Z. | Waypoint-Panel: `tabCount: 5` (ein Tab je Act) |
| `data/global/ui/layouts/waypointspaneloriginalhd.json` | 4 411 | – | dito Classic/LoD (4 Tabs) |

### 2.2 **Nicht** im Repo (Basisspiel) – würde man brauchen

[BELEGT, dass sie fehlen; Inhalt/Funktion teils ANNAHME]
- `lvltypes.txt` – Level-Typen: ordnet jedem `LevelType` (0–35, `levels.txt` Spalte 29) die DT1-Tile-Dateien zu; `automap.txt` ist danach benannt (35 Typnamen: `1 Town` … `5 Lava`).
- `lvlsub.txt` – Substitutions-/Outdoor-Bausteine für `DrlgType 3` **[ANNAHME]**.
- `objgroup.txt` – Objektgruppen (`ObjGrp0–7` in `levels.txt`) **[ANNAHME]**.
- **Alle `.dt1`-Tile-Dateien** (Boden/Wand/Objekt-Kacheln) – im Repo gibt es null.
- **~1 970 von 2 044** in `lvlprest.txt` referenzierten `.ds1`-Dateien (nur 77 liegen im Repo; geprüft, Pfad-Vergleich ohne Groß-/Kleinschreibung). Beispiel: `lvlprest.txt:1094` → `Garden/ColossalSummit/ColossalSummit.ds1` – kein `Garden/`-Ordner im Repo.
- HD-Presets der unveränderten Basisspiel-DS1 (nur 63 von 82 Repo-DS1 haben eines; 19 geänderte Basis-DS1 ohne Preset, z. B. `act1/graveyard/gravey.ds1`).
- Sämtliche Sprites/Sounds/Modelle der Basisspiel-Gebiete.
→ **Ein neues Gebiet kann sich ausschließlich auf Basisspiel-Assets stützen, ohne dass sie im Repo liegen; man referenziert sie nur per Pfad/Id.**
  Praktische Folge: Ob ein referenzierter DS1-Pfad existiert, lässt sich aus dem Repo allein **nicht** prüfen.

---

## 3. `levels.txt` im Detail

### 3.1 Struktur [BELEGT]
- Header `levels.txt:1`, **188 Spalten**, Datenzeilen `2–140` (139 Zeilen). Zeile 2 = `Null` (Id 0). **Zeile 111 = `Expansion`** (reine Trennzeile ohne Id, `levels.txt:111`).
- Zeilenenden: `levels.txt` **CRLF**, `base/levels.txt` **LF**, kein BOM, Datei endet mit `\n`.
- Spaltengruppen (Nummer = 1-basierte Spalte):

| Spalten | Felder |
|---|---|
| 1–3 | `Name` (Kommentar/Bezeichner, eindeutig), `*StringName` (Kommentar, wird ignoriert), **`Id`** |
| 4–7 | `Pal`, **`Act`** (0–4), `QuestFlag`, `QuestFlagEx` |
| 8–17 | `Layer`, `SizeX/Y` (+`(N)`,`(H)`), `OffsetX/Y`, `CompletionTotalRoomsOverride` |
| 18–27 | `Depend`, `Teleport`, `Rain`, `Mud`, `NoPer`, `LOSDraw`, `FloorFilter`, `BlankScreen`, `DrawEdges`, `*IsInside` |
| **28–33** | **`DrlgType`** (1 Maze, 2 Preset, 3 Outdoor – siehe 3.3), **`LevelType`**, `SubType`, `SubTheme`, `SubWaypoint`, `SubShrine` |
| **34–41 / 42–49** | `Vis0–7` (Ziel-Level-Ids) / `Warp0–7` (Zeile in `lvlwarp.txt`, −1 = keine) |
| 50–58 | `Intensity/Red/Green/Blue`, `Portal`, `Position`, `SaveMonsters`, `Quest`, `WarpDist` |
| 59–76 | `MonLvl*`, `MonLvlEx*`, `MonDen*`, `MonUMin/Max*`, `MonWndr`, `MonSpcWalk`, `NumMon` |
| 77–164 | `mon1–25`, `rangedspawn`, `nmon1–25`, `umon1–25`, `cmon1–4`, `cpct1–4`, `camt1–4` (Monsterlisten Normal/Nightmare/Hell) |
| 165–170 | `Themes`, `SoundEnv`, **`Waypoint`** (0–39, 255 = keiner), **`LevelName`**, **`LevelWarp`**, **`LevelEntry`** (String-Keys → `levels.json`) |
| 171–186 | `ObjGrp0–7`, `ObjPrb0–7` |
| **187–188** | **`LevelGroup`** (→ `levelgroups.txt` `*Name`), **`PreventTownPortal`** |

- **Das Spaltenlayout wandert mit Blizzard-Patches** [BELEGT per Header-Vergleich in der Historie]: 186 Spalten bis `cf4c114e`; `PreventTownPortal` kam am Ende dazu im RotW-Import `11aa02dc` (→ 187); **`CompletionTotalRoomsOverride` wurde in `799a67d9`** („Blizz Patch Compatibility“, 2026-05-23) **mitten in die Tabelle (Spalte 17) eingefügt** (→ 188) und verschob damit alle folgenden Spalten. → Neue Zeilen **immer per Spaltenname** schreiben (Header lesen), nie nach festen Positionen; sonst bricht der nächste Blizzard-Patch jede handgebaute Zeile.

### 3.2 Ids [BELEGT]
- Ids **0–137, lückenlos, ohne Duplikate** (138 Werte; geprüft per Skript). **Höchste Id = 137, nächste freie = 138.** Es gibt keine freien Lücken.
- Act-Bereiche (aus Spalte `Act`/Namen): Act 1 = 1–39 (39 = Moo Moo Farm, `levels.txt:41`), Act 2 = 40–74, Act 3 = 75–102, Act 4 = 103–108, Act 5 = 109–137 (133–136 = Pandemonium 1–3 + Finale, 137 = Colossal Summit).
- **Physischer Zeilenindex ≠ Id** ab Id 109 wegen der `Expansion`-Trennzeile (Id 109 liegt auf Datenzeile 110, 0-basiert). Das SDK bestätigt: `findRowById` vs. `getRow` (`PluginSDK/README.md:1309-1311`) und Beispiel „level 124 is row …“.
- `DrlgType`-Verteilung: 70× Typ 1, 36× Typ 2, 31× Typ 3, 1× Typ 0 (Null).
  **Aus den Daten bewiesen:** alle 70 Typ-1-Level haben eine Zeile in `lvlmaze.txt`; alle 36 Typ-2-Level haben eine Zeile in `lvlprest.txt` (Spalte `LevelId`); Typ-3-Level haben weder noch (31/31) – deren Layout entsteht aus den 1056 `lvlprest`-Zeilen mit `LevelId = 0` (benannte Bausteine) **[ANNAHME zur Mechanik]**.
  `LevelType` (Spalte 29) hat max. **35** (verweist auf das fehlende `lvltypes.txt`).
- `OffsetX/OffsetY` positionieren Level im Welt-Koordinatenraum je Act; Act 5 nutzt Spalten X = 1000, 2000, 2500, 3000, 4000, 5000 (Y stapelt sich nach Größe). Max. vorkommender Offset: X 5000, Y 3100. `Layer` ist **nicht** eindeutig (Wiederverwendung belegt: Id 136 → Layer 25 wie Tristram, 133–135 → 11/28/92).

### 3.3 Was sind Reimagined-Änderungen? [BELEGT über Git, nicht über Vanilla-Vergleich]
Eine Vanilla-`levels.txt` liegt **nicht** im Repo; ein direkter Diff gegen Vanilla ist nicht möglich. Erkennbar über die Historie:

| Änderung | Beleg |
|---|---|
| Moo-Moo-Farm bekommt Waypoint 39 (Act-4-Untermenü): `levels.txt:41` + `actinfo.txt:5` (Act 4 → `waypoint4 = Act 1 - Moo Moo Farm`); passend dazu geänderte `bivouac.ds1` + HD-Preset | `852c28a1` (2026-03-05), `5ead2711` |
| Monsterdichte Act 1 früh −25 % | `d71f4115` (2024-05-09) |
| Monster-/Gebietslevel-Tweaks | `e25466cf`, `788f6af1` („fix area levels“: 60 Zeilen je Bank, Spalten im Bereich der Monsterlevel; **keine** Id-Änderung) |
| Terror-Zonen-Abdunklung entfernt; `levelgroups.txt` neu ins Repo | `890629d1` (2026-04-07) |
| **Komplett eigene Gebiete** (heute entfernt): Pitzilla (137–139), Panda-Portal-Level, Forsaken Labyrinth 00–10 (Ids 137–155) | `7f0ed3b3` (2025-08-03), entfernt in `890629d1` |
| Id 136 „Pandemonium Finale“: Klon von Tristram (Id 38) mit anderen Monster-/Level-Werten, `PreventTownPortal=1` | `levels.txt:139` vs. `:40` |

Zusätzlich: Reimagined hat zuletzt für einen Loader-Plugin-Anker (`StashSearchAnchor`) im Layout `bankexpansionlayouthd.json` gesorgt (`86aae80e`) – das Projekt ist also bereits im D2RLoader-Umfeld unterwegs; es gibt aber **keinerlei** `d2rloader`-Referenzen im Repo (grep leer).

---

## 4. Vollständige Kette eines Beispielgebiets

### 4.A Aktuelles, aktives Beispiel: **Act 5 – Colossal Summit** (Id 137)

Kleinstes aktives Gebiet: **1 Level, keine Warps, keine Waypoints, Zugang nur per Cube-Portal.**
[ANNAHME] Es ist **Blizzard-/RotW-Inhalt**, nicht von Reimagined erstellt. Indizien [BELEGT]: erstmals im RotW-Import (`11aa02dc` / Daten-Import-Commits `c06a3112`, `f38f19f7`, `fa423799` vom 2026-02-12/13), DS1-Pfad `Garden/…` liegt nicht im Repo, Strings haben Ids < 30000 (Blizzard-Bereich) und sind in allen 13 Sprachen übersetzt.

| # | Glied der Kette | Datei:Zeile | Inhalt |
|---|---|---|---|
| 1 | **Zugang** (Portal erzeugen) | `cubemain.txt:688` | Rezept „Uber Ancient Summon Material Act 1–5 → Portal to Colossal Summit“: `op=28` („Special“, lt. `lint/config.json` `cubeOps`), `numinputs=5`, Inputs `ua1…ua5`, Output **`Red Portal,lvl=137,qty=1`**, `min diff=2` |
| 2 | Zutaten | `misc.txt:159-163` | Items `ua1`–`ua5` „Uber Ancient Summon Material Act 1–5“ (Typ `ques`) |
| 3 | Zutaten-Drops | `treasureclassex.txt:63-67` | TCs „Ancient Statue 1–5“ droppen `ua1…ua5` |
| 4 | **Level-Definition** | `levels.txt:140` | `Id=137`, `Act=4`, `Layer=87`, `Size 20×28`, `Offset 5000,2000`, `DrlgType=2` (Preset), `LevelType=31`, `Vis0–7=0`, `Warp0–7=-1`, `Waypoint=255`, `LevelName=ColossalSummit`, `LevelWarp=ToTheColossalSummit`, `LevelEntry=EnteringTheColossalSummit`, `LevelGroup=Act 5 - Colossal Summit`, `PreventTownPortal=1`, `NumMon=3` aber `mon1…` leer, `MonLvl 27/52/77` |
| 5 | **Preset/Layout-Zuordnung** | `lvlprest.txt:1094` | `Def=1091`, `LevelId=137`, `Populate=1`, `KillEdge=1`, `FillBlanks=1`, `Scan=1`, `Files=1`, `File1=Garden/ColossalSummit/ColossalSummit.ds1`, `Dt1Mask=29360148` (=`0x1C00014`, identisch mit `MtnTop`, `lvlprest.txt:1092`) |
| 6 | **Layout (`.ds1`)** | **nicht im Repo** | → Basisspiel **[ANNAHME]**; keine HD-Preset-JSON im Repo → Basisspiel-Preset |
| 7 | Tiles (`.dt1`) | nicht im Repo | via `LevelType=31` → `lvltypes.txt` (Basisspiel) |
| 8 | **Gruppe** | `levelgroups.txt:102` | `Act 5 - Colossal Summit`, `CompleteThreshold=100`, `NameString=ColossalSummit` |
| 9 | **Strings** | `levels.json:7127 / 7144 / 7161` | Keys `ColossalSummit` (id 27867), `ToTheColossalSummit` (27868), `EnteringTheColossalSummit` (27869), je 13 Sprachen |
| 10 | **Automap** | `automap.txt` (Typ `5 Barricade`, erste Zeile `:2746`) | **keine eigenen Zeilen** – Automap kommt über `LevelType=31` |
| 11 | **Monster** | `monstats.txt:748-750`, `monstats2.txt:620-622` | `colossal1–3` (Ids 745–747, „Ancient Barbarian 1–3“, AI `ColossalWhirlwind/Throw/Leap`). **Nicht** über `levels.txt` verdrahtet (Monsterlisten leer) → vermutlich über DS1-Platzhalter **[ANNAHME]** |
| 12 | **Objekte** | `objects.txt:586-589` | `ColossalAltar`, `colossal_statue_1–3` |
| 13 | Skills/Missiles | `skills.txt:415-416`, `missiles.txt:733-743` | „Colossal Volcano/Thunder Storm“, `colossal*`-Geschosse |
| 14 | **Waypoint/Warp** | – | keiner: `Waypoint=255`, kein `lvlwarp`-Eintrag, kein anderes Level verweist per `Vis` darauf |
| 15 | Terror-Zone | `desecratedzones.json` | Id 137 **nicht** enthalten (höchste `level_id` = 132) |

Lesehilfe: Nur die Glieder 1, 4, 5, 8, 9 sind **minimal nötig** für ein Gebiet, das (a) im Spiel existiert und (b) per Portal erreichbar ist; die übrigen sind Inhalt (Monster/Objekte) oder Zugang.

### 4.B Historisches Beispiel mit **vollständig im Repo-Verlauf vorhandenen Layouts**: **Pitzilla-Höhle** (Ids 137–139)

Von Reimagined selbst gebaut; durch `git show 7f0ed3b3^:<pfad>` vollständig rekonstruierbar. Zeigt die Kette **Maze-Level → Preset-Level → Preset-Boss-Level → Cube-Portal**:

| Glied | Quelle (Stand `7f0ed3b3^`) | Inhalt |
|---|---|---|
| Level 137 `Pitzilla 1` | `levels.txt:140` | `DrlgType=1` (Maze), `LevelType=3`, `Layer=5`, `Size 200×200`, `Offset 200,200`, `Vis0=109` (Harrogath) `Warp0=83` („Null Warp“), `Vis4=138` `Warp4=5`, `NumMon=3` (`zombie1…`), `MonLvl 28/53/78`, `MonDen=600`, Strings `Pitzilla Level 1` / `To The …` / `EnteringThePitzillaLevel1` |
| Level 138 `Pitzilla Boss Room 1` | `levels.txt:141` | `DrlgType=2` (Preset), `Size 24×24`, `Vis1=137 Warp1=4`, `Vis2=139 Warp2=84` |
| Level 139 `Pitzilla End` | `levels.txt:142` | `DrlgType=2`, `Vis1=138 Warp1=4` |
| Maze-Zeile | `lvlmaze.txt` | `137 → Rooms 4/4/4, 24×24, Merge 500` |
| Preset-Zeilen | `lvlprest.txt` | `Def 1091 / LevelId 138 / File1 Act1/Caves/pzboss1.ds1 / Dt1Mask 1`; `Def 1092 / LevelId 139 / Act1/Caves/pzend.ds1 / Dt1Mask 1` |
| Warp-Zeile | `lvlwarp.txt` | `83 Null Warp`, `84 Pitzilla Boss Down` |
| Layouts | `data/global/tiles/act1/caves/pzboss1.ds1` (11 387 B), `pzend.ds1` (11 506 B) | **liegen heute noch im Repo, aber unreferenziert**; **kein** HD-Preset-JSON dazu |
| Zugang | `cubemain.txt:689-690` (heute `enabled=0`) | „Town Portal Scroll & Snapchip's Soul & (Stacked) Skull = Pitzilla Red Portal“, Output `Red Portal,qty=137` |

**Merke:** Diese Rezepte zeigen noch auf Id 137 – die heute Colossal Summit ist. Bei Reaktivierung würden sie in das falsche Gebiet führen: ein **konkretes Beispiel für Id-Drift** (siehe R1).

### 4.C Historisches Großbeispiel: **Forsaken Labyrinth** (Ids 145–155; Commit `7f0ed3b3`)
Änderungen im Commit (13 Dateien): `levels.txt` (+Level), `lvlmaze.txt` (+8 Maze-Level), `lvlprest.txt` (+3 Presets, `Labyrinth00/05/10.ds1`), `lvlwarp.txt` (Warp 83 „Act 4 Town to Forsaken Labyrinth“), **geänderte Stadt `townwest.ds1`** (Tor/Warp platziert) + `townwest.json`, 3 neue HD-Presets (`labyrinth00/05/10.json`, 0,7–2,8 MB), `levels.json` (je Level 3 Keys; Ids ab 95000; 12 Fremdsprachen = englischer Text). `Harrogath` (Id 109) bekam `Vis0=145`, `Warp0=83`.
Heute (nach `890629d1`): Level-Zeilen entfernt, Harrogath `Vis0=0 / Warp0=-1`; **übrig geblieben** sind `lvlwarp.txt:91`, die 3 DS1 und 3 HD-Presets (unreferenziert).

---

## 5. `scripts/` und `lint/`

### 5.1 `scripts/` [BELEGT]
| Datei | Tut |
|---|---|
| `copy-files.bat` | spiegelt `data/` per `robocopy /MIR` nach `…\mods\Reimagined\Reimagined.mpq\data` (hartcodierte Windows-Pfade) |
| `data/global/excel/a_copy_excel_base.bat` | kopiert `excel/*.txt` → `excel/base/` (außer `treasureclassex.txt`), überspringt unveränderte |
| `fill-in-ids.js` | **nummeriert String-JSON-Ids ≥ 30000 neu durch**, je Datei ab fixem Start: `item-modifiers 48000`, `item-nameaffixes 49000`, `item-names 50000`, `item-runes 53500`, **`levels 54000`**, `monsters 55000`, `skills 56000`, `ui 57000`. Ids < 30000 bleiben unberührt (Blizzard-Bereich) |
| `make_levels_table.py` | HTML-Tabelle aus `levels.txt` (Spalten Name, `*StringName`, `MonLvlEx*`) fürs Wiki |
| `compare-key-entries.js` | listet Keys, die in einem Vanilla-`ui.json` (`../../cascs/…`, nicht im Repo) vorkommen, aber in `ui.json` fehlen |
| `copy-english-names.js` | füllt leere Übersetzungen mit `enUS` – **hartcodiert auf `item-names.json`** |
| `translate_items.py` | DeepL-Übersetzung fehlender Einträge in `item-names.json` |

### 5.2 `lint/` [BELEGT]
- `d2rlint.exe` (95,9 MB, **Windows-PE**), `config.json`, `output.txt`/`output_base.txt` (Baseline vom 2026-08-23).
- Zwei Läufe (`config.json:10`): Workspace `..\data\global\excel` (exkl. `base`, Log `output.txt`) und `..\data\global\excel\base` (Log `output_base.txt`).
- Level-relevante Regeln (alle `warn`): `Level/ValidWarp` (`:64`), `Level/ValidWPs` (`:67`), `Basic/LinkedExcel` (`:34`, Querverweise zwischen Tabellen), `Basic/ExcelColumns`, `Basic/NumericBounds`, `Basic/StringCheck`, `Json/DuplicateIds`, `Json/StringFormat`, **`Json/KeyUsage` mit `idStart: 30000`** (`:94-96`: Strings ab Id 30000 müssen von einer `.txt` oder einem Layout-JSON referenziert werden).
- Baseline: ausschließlich 28× `Monsters/ValidChains` + 1× `Json/KeyUsage` – **null** Level-Warnungen. Ein neues Gebiet sollte die Baseline nicht verschlechtern.
- Genaue Semantik der Regeln ist nicht im Repo dokumentiert **[ANNAHME]**: `ValidWarp` ≈ `Vis`/`Warp`-Konsistenz, `ValidWPs` ≈ Waypoint-Zuordnung (`actinfo`).

### 5.3 Was für neue Gebiete zu erweitern wäre
1. **Kein Skript prüft Level-Konsistenz außerhalb des (nur Windows) Linters.** Empfehlung (Phase 2, *neue* Datei): `scripts/check_levels.py` – prüft Id-Eindeutigkeit/-Lückenlosigkeit, `levels`↔`lvlprest`/`lvlmaze`, `LevelGroup` ∈ `levelgroups`, String-Keys in `levels.json` (13 Sprachen), `excel` == `excel/base`, Spaltenanzahl.
2. `fill-in-ids.js` deckt `levels.json` bereits ab (ab 54000) – neue Gebiets-Strings gehören in diesen Block. **Achtung:** Das Skript nummeriert alle Ids ≥ 30000 neu; eingefügte Strings daher ans Ende hängen und dann lauffähig halten.
3. `copy-english-names.js` ist hartcodiert auf `item-names.json`; für `levels.json` müsste man den Pfad parametrisieren (oder Übersetzungen manuell eintragen).
4. `copy-files.bat` braucht nur dann Anpassung, wenn neue Ordner (z. B. `data/global/tiles/<neu>/`) außerhalb von `data/` liegen – tun sie nicht. Es müsste **kein** Installationsskript erweitert werden.
5. `a_copy_excel_base.bat` ist **Windows-only**; unter Linux muss die Spiegelung `excel/` → `excel/base/` (CRLF→LF bei `levels.txt`/`cubemain.txt`!) selbst erledigt werden.

---

## 6. D2RLoader PluginSDK (`../PluginSDK`, geklont ✔)

Klonen funktionierte ohne Netzwerkprobleme.

### 6.1 Versionen, Anforderungen, Laden [BELEGT]
- **SDK 0.3.0, Plugin-ABI 4**, Minimum-ABI 2 (`include/D2RLPlugin/version.h`; `README.md:10-16`). ABI 3 führte Rollen ein (`README.md:288-290`), ABI 4 den HTTPS-Service (`version.h`, Kommentar zu `D2RL_PLUGIN_HTTP_ABI_VERSION`). ABI 2/3 werden weiter akzeptiert.
- Windows x64, CMake ≥ 3.29, MSVC oder clang-cl, C++20 (`README.md:20-23`, `CMakeLists.txt`). Die SDK ist Header-only (INTERFACE-Library) + 14 Beispiele.
- **D2R-Version:** das SDK nennt keine feste D2R-Build-Nummer. Hinweise: *Reign-of-the-Warlock*-Banken, „retail 3.2“ (`README.md:1343,1352`), „D2RLoader 1.3.1“ (`README.md:1183`). Der Loader reicht `buildVersion`, `buildName`, `exeBase`, `modDataVersionBuild` an das Plugin (`context.h:132-147`) → Build-Prüfung zur Laufzeit ist vorgesehen.
- **Laden:** global `<game>/d2rloader/plugins/`, **mod-spezifisch** `<game>/mods/<mod>/d2rloader/plugins/`; Mod-Plugins laden zuerst und ersetzen gleichnamige globale (`README.md:170-185`). Flag `ModScopedOnly` möglich. Jedes Plugin braucht genau **eine Rolle** `Client` / `Server` / `Shared` (`lifecycle.h`, `README.md:187-197`). **Shared/Server-Plugins müssen in TCP/IP bei Host und Client übereinstimmen** (Id, Version, ABI, Scope) und landen in der Charakter-Historie.
- Pflicht-Exports: `D2RLoaderGetPluginInfo`, `D2RLoaderLoadPlugin` (optional `D2RLoaderUnloadPlugin`); Manifest-Resource (RCDATA = ABI) (`examples/README.md`, „Plugin manifest“).
- Mod-Metadaten/Strings: `d2rloader/metadata.json` mit `namespace`; Plugin-Strings in `data/local/lng/strings/d2rloader/<plugin-id>/strings.json`; Ressourcenreihenfolge „aktive Mod → Mod-Plugin → globales Plugin → D2RLoader → CASC/Basisspiel“ (`README.md:570`, `LOCALIZATION.md`).

### 6.2 API-Überblick [BELEGT]
21 Services (`services.h`): Lifecycle, Resource, CustomTable, Panel, Inventory, Network, Input, DataTable, SharedEvent, Diagnostics, GameRule, Widget, Thread, Localization, Item, Http, ItemInteraction, **Encounter**, **Mutation**, Overlay, PluginCommunication.

| Kategorie | Was geht |
|---|---|
| **Speicher-Patches** | `PluginContext::PatchBytes/Fill/Nop/Write*/PatchJmpRel32/PatchCallRel32` (jeweils mit *expected bytes* + RVA relativ zu `D2R.exe`); atomare Transaktionen `MutationService` (Patches, Rel32, Inline-Hooks); Diagnose `DiagnosticsService` (wer hat Bytes geändert). DLL-lose JSON-Patches in `<game>/(mods/<mod>/)d2rloader/patches/*.json` (`examples/json-patches/`). Inline-Hooks nur mit Flag `NativeHooks`; Rel32-Ziele müssen **im** `D2R.exe`-Image liegen (`README.md:1178-1183`). |
| **Events** | `LifecycleService`: `DataTablesLoadedEvent` (nach jedem Tabellen-(Neu)Laden), GameJoined/GameLeft, LocalPlayerReady, **ActChanged, LevelChanged** (Level-Id), PlayerResurrected, PlayerLevelChanged, QuestCompleted, **MonsterDeath** (mit `levelId`) (`lifecycle_events.h:30-45`, `:127`, `:146`); `SharedEventService` (Tooltip/UI-Nachrichten); `ItemInteractionService`. |
| **Tabellenzugriff** | `DataTableService`: **read-only**, kompilierte Tabellen je Bank (`Classic/Lod/Rotw`), nur im Game-Thread, Zeiger verfallen beim nächsten Laden (`data_tables.h:117-118,196`). Relevante `TableId`s: **`Levels=48, LevelGroups=49, LevelDefs=50, LvlPrest=51, LvlTypes=67, LvlWarp=68, LvlMaze=69`**, außerdem `MonStats`, `Objects`, `CubeMain`, `ActInfo`, … `findRowById` für `Levels`/`Skills` (logische Id). `CustomTableService`: **plugin-eigene** TXT/BIN-Tabellen unter `d2rloader/<plugin-id>/…` (max. 349 Spalten). |
| **UI** | `PanelService` (eigene Panels + Layout-JSON, `registerChildLayout` für Inventar-Panel), `WidgetService` (vorhandene Widgets suchen/steuern), `OverlayService` (nur Anzeige: Linien/Rechtecke/Text), `InputService` (Aktionen im Controls-Menü), Controller-Routen. |
| **Controls/Console** | Konsolenbefehle (`RegisterConsoleCommand`), `ExecuteConsoleCommand` (Core-Export Ordinal 14: schaltet kurz das Cheat-Flag, sendet Befehl); `PrepareHostEnvironment` für Dedicated Server. |
| **Gameplay-Dienste** | `ItemService` (Items erzeugen/ändern/tauschen), `InventoryService` (eigene Spielerseite), `GameRuleService`, `NetworkService` (private Plugin-Nachrichten, ≤ 224 B), `HttpService`, `ThreadService`, `EncounterService` (Herald-/Boss-Entscheidungen inkl. `levelId`, Modifier-Policy). |
| **Low-Level** | Core-Exports Ordinal 1–99, **Reimplementation-Exports 2000–2999**: nur `GetLevelIdFromUnit`, `GetLevelIdFromRoomContext` (lesend), Item-/Cast-Helfer (`reimplementation_exports.h`). |
| **Ressourcen** | `ResourceService`: Dateien aus dem Speicher registrieren, **nur** unter `data/.../d2rloader/<plugin-id>/…` bzw. `data/global/ui/layouts/<plugin-id>/…`; max. 64 MiB je Datei, 256 MiB je Ladevorgang (`resources.h`). Companion-MPQ `d2rl-<plugin-id>.mpq` neben der DLL. |

### 6.3 Gibt es etwas, das Level-Generierung, Level-Tabellen oder Datenladen beeinflusst?
- **Level-Generierung: nein.** Suche über alle Header nach `level|drlg|preset|ds1|dt1|area|maze|lvlprest|lvlwarp|waypoint|portal` liefert nur Level-Ids in Events/Encounter-Strukturen, die lesende `GetLevelIdFrom*`-Helfer und die `TableId`-Einträge. **Kein** Service erzeugt/ersetzt Level, öffnet Portale, teleportiert Spieler oder registriert DS1/DT1.
- **Level-Tabellen:** nur lesen (`DataTableService`); nichts schreibt `levels.txt` & Co. Stock-Tabellen ersetzen: **nein** – Plugins sehen nur ihren eigenen `d2rloader/<plugin-id>/`-Pfad (`README.md:580-584`); die Mod (`Reimagined.mpq`) bleibt die Quelle der Stock-Tabellen.
- **Datenladen:** Plugins können nach dem Laden reagieren (`DataTablesLoadedEvent`), eigene Tabellen mitladen lassen und eigene Strings per Namespace einbringen (`<namespace>:Key`, `README.md:603-605`). Mod-Strings können Plugin-Texte überschreiben, nicht umgekehrt Stock-Strings.
- **Einzige theoretische Hintertür:** rohe Inline-Hooks (`NativeHooks`) auf D2R-interne Funktionen (z. B. Level-Aufbau) – **[ANNAHME]** technisch möglich; das SDK liefert dafür **keine Offsets, Symbole oder Strukturen**; jede RVA ist build-spezifisch und müsste per Reverse-Engineering ermittelt werden.

### 6.4 Einschränkungen [BELEGT]
- Build-spezifisch: RVAs/Expected-Bytes (`patching.h`, `examples/patching`: z. B. `GetVisibleLineCountRva = 0x000A8950`), kompilierte Tabellenzeilen („use the compiled layout for the running game build“, `README.md:1319`), Service-Größenprüfung pro Feld (`HasXField`). Jedes D2R-Update kann Hooks/Layouts brechen.
- Nur Windows x64; Plugin-Rollen/Multiplayer-Match-Pflicht für Shared/Server; Battle.net: `NetworkService` nicht verfügbar (`README.md:1015`).
- D2RLoader selbst (Hostprogramm) ist **nicht** Teil des SDK-Repos; was der Loader sonst kann, ist hier nicht belegbar.

### 6.5 Bewertung für „neue Gebiete“
Was das Plugin sinnvoll **ergänzen** könnte (alles optional, kein Muss): per `LevelChanged`/`MonsterDeath` Gebietslogik (Boss-Kill-Flags, Belohnungen, Kommunikation), `EncounterService` für Boss-Modifier im neuen Gebiet, `ItemService` zum Ausgeben von Portal-Schlüsseln, eigene Panels/Overlay (z. B. Gebietsanzeige). Zum **Erzeugen** des Gebiets selbst trägt es nach heutiger API **nichts** bei.

---

## 7. Harte Engine-Grenzen – nur was belegbar ist

[BELEGT aus den Dateien]
| Grenze | Beleg |
|---|---|
| **5 Acts**: `Act` ∈ 0–4; Waypoint-Panel `tabCount: 5` | `levels.txt` (Spalte 5), `waypointspanelexpansionhd.json:20-24` |
| **Waypoints**: Index 0–39 (40 vergeben), **max. 9 pro Act** laut Tabellenschema `actinfo.txt` (`waypoint1–9`). Act 1, 2, 3, 5 sind **voll (9/9)**, Act 4 nutzt 4 (inkl. Moo Moo Farm) | `actinfo.txt:2-6`, `levels.txt`-Spalte 167 |
| **Verbindungen je Level**: 8 (`Vis0–7`/`Warp0–7`) | `levels.txt` Spalten 34–49 |
| **Monsterlisten**: 25 je Schwierigkeit + 4 Champions | `levels.txt` Spalten 77–164 |
| **Objektgruppen**: 8 | Spalten 171–186 |
| **DS1-Varianten je Preset**: 6 (`File1–6`) | `lvlprest.txt` Spalten 16–22 |
| **Level-Typen**: 0–35 (35 Typnamen in `automap.txt`) | `levels.txt` Spalte 29, `automap.txt` |
| **Ids**: Id-Spalte explizit; 0–137 belegt; **156 Level (Ids 0–155) waren in einer ausgelieferten Reimagined-Version belegt** (`7f0ed3b3`) – die Obergrenze liegt also mindestens dort | Git-Historie (ob in-game getestet: nicht prüfbar) |
| **Custom-Tabellen im SDK**: ≤ 349 Spalten, 1 MiB/Zeile | `custom_tables.h` |
| Terror-Zonen kennen nur die in `desecratedzones.json` aufgeführten `level_id`s (≤ 132) | `desecratedzones.json` |

**[EXTERN BELEGT – Phase 1b, nicht selbst verifiziert]** Community-Reverse-Engineering der RuffnecKk-Suite (`RuffnecKk-D2RLoader-Suite/research/d2r/findings.md:3332-3480`, `known-rvas.json`; getestet auf D2R 3.3.93847 / 3.2.92777; Quellen dort selbst als „Static findings and historical runtime observations prove only their documented scope“ markiert). Sie präzisieren und **korrigieren Annahmen dieses Berichts**:
| Grenze | Aussage der Suite-Research |
|---|---|
| **Max. 1023 Level-Zeilen** (kanonische, zusammenhängende Ids 0–1022) | Vergleich `cmp [Levels+0x1468], 0x400` an RVA `0x330446`; Classic/LoD-Bank hatte dort 137 Zeilen, RotW 147 |
| **Act wird aus der Level-Id über ActInfo-Bereiche aufgelöst** | `DRLG_ResolveActFromLevelId` (RVA `0x326710`) läuft `ActInfo`-Records rückwärts und vergleicht die Level-Id mit `rangeStart`; D2MOO 1.10f nutzte feste Schwellen `{1,40,75,103,109,1024}`. **Folge:** Ein neues Level landet automatisch im letzten Act (Act 5), solange seine Id ≥ 109 ist; ein neues Level in Act 1–4 mit Id > 137 würde als Act 5 aufgelöst. Die Suite-Plugin `extended-act-level-ids` („Allows new levels to belong to any act“) patcht genau das – laut Audit-Dokument ist es aber aus dem aktiven Katalog entfernt (`research/d2rloader-1.3-public-audit.md:16`), Status unklar. |
| **Town Portal / Red Portal in Level-Ids > 255 crasht (Assertion)** | `BC_ASSERT: eLevelIdLocal <= 255` in `D2GAME_CreateLinkPortal` (RVA `0x435DD0`); Portal-Zustand (`InteractType`) ist 1 Byte. **Folge für Phase 2:** neue Level-Ids bei ≤ 255 halten (Id 138 ist unkritisch). |
| Sichtbarkeits-Pakete `0x07/0x08` transportieren Level-Id nur als Byte | ebenfalls ab Id 256 problematisch (Suite löst das mit eigenem Codec) |

→ Praktisch gilt für neue Gebiete: **Act 5, Id 138–255** ist der belegt unkritische Bereich; alles andere braucht Engine-Patches.

[ANNAHME – nicht belegt]
- Ob Lücken in der Id-Folge erlaubt sind (die Suite-Research spricht von „zusammenhängenden“ Ids `0..1022`; die Id-Spalte ist explizit, aber der kompilierte Zeilenindex ≠ Id).
- Dass ein Level mit anderer `Act`-Zuordnung als sein Zugangs-Level problemlos ist (Pandemonium 133–135 sind Act 5, nutzen aber Act-1/2/5-Level-Typen – gespielt aus Act 5 heraus, `levels.txt:136-138`). Zur Act-Auflösung siehe oben.
- Rolle von `Layer`, `Position`, `Depend`, `Teleport` und genaue Semantik von `OffsetX/Y` (nur Nutzung beobachtbar).
- `DrlgType`-Bedeutung 1/2/3 = Maze/Preset/Outdoor (aus den Tabellenzuordnungen eindeutig abgeleitet, Namen aber Allgemeinwissen).
- DS1-Format/Editoren, HD-Preset-Fallback bei fehlendem JSON, Wirkung von `combined_ds1.bin`.

---

## 8. Risiken und offene Fragen

### Risiken
| # | Risiko | Einstufung | Beleg / Gegenmaßnahme |
|---|---|---|---|
| **R1** | **Id-Kollision mit Blizzard-Updates** | hoch | Vor dem RotW-Import war Id 137 ein Reimagined-Level (Pitzilla 1, später „Panda – Dark Tristram“); im RotW-Import (`11aa02dc`) wurde **137 = Colossal Summit** (Blizzard) und der gesamte Reimagined-Block rutschte von 137–155 auf **138–156** (belegt: Id 146 = „Forsaken Labyrinth 00“). **Verweise außerhalb von `levels.txt` wurden dabei nicht mitgezogen:** Harrogath behielt `Vis0=145` (zeigt danach auf „Palace of the Degenerate“ statt auf Labyrinth 00 = 146), und `cubemain.txt:689-690` zeigen noch auf `qty=137`. Der Block wurde später gelöscht (`890629d1`). Gegenmaßnahme: **alle** Verweise auf eine Level-Id (levels `Vis`, `lvlprest.LevelId`, `lvlmaze.Level`, Cube-Rezepte `lvl=…`, `desecratedzones.json`, `actinfo`) zentral dokumentieren (Id-Registry) und per Check-Skript prüfen. |
| **R2** | **Merge-Konflikte** mit Reimagined-Updates | mittel–hoch | `levels.txt` (CRLF, 188 Spalten, ~80 kB) wird oft angefasst (28 Commits); neue Zeilen am **Dateiende** (nach Zeile 140) sind der konfliktärmste Ort, aber Blizzard-Spaltenänderungen (186→187→188, davon eine **Einfügung mitten in der Tabelle**, `799a67d9`) betreffen jede Zeile – dann müssten auch handgebaute Zeilen nachgezogen werden. |
| **R3** | Zwei Excel-Bänke / Zeilenenden | mittel | `excel/` ≡ `excel/base/`; `levels.txt` & `cubemain.txt` CRLF, `base/` LF (Zeilenenden pro Datei erhalten, sonst Voll-Diff). |
| **R4** | **DS1/HD-Preset-Erstellung** | hoch (für eigene Layouts) | Binärformat + 0,2–5 MB HD-JSON je DS1; Werkzeuge nicht im Repo; ohne Spiel nicht prüfbar. Wiederverwendung vorhandener Basisspiel-DS1 umgeht das (Präzedenz: Id 136 mit `Tri_Town4.ds1`, kein eigenes Preset im Repo). |
| **R5** | Zugang über die Stadt | mittel–hoch | Ein Tor in Harrogath erfordert Änderung von `townwest.ds1` (76 kB binär) + `townwest.json` (5,1 MB) + `lvlwarp.txt` – Datei wurde schon von `7f0ed3b3`/`11aa02dc` geändert → hohes Konfliktrisiko. **Cube-Portal vermeidet das** (Colossal/Pitzilla-Muster). |
| **R6** | Waypoints | mittel | Act 5 hat 9/9 Slots; ein Waypoint bräuchte einen anderen Act (Muster Cow-WP in Act 4, `852c28a1`). |
| **R7** | Terror-Zonen/Level-Gruppen | niedrig–mittel | Neue Level brauchen eine `levelgroups.txt`-Zeile (Muster Id 137); Aufnahme in Terror-Zonen = Änderung an `desecratedzones.json` (optional). |
| **R8** | Balance/Monster | niedrig | Monsterlisten/-level pro Schwierigkeit (`mon/nmon/umon`, `MonLvl*`). |
| **R9** | Multiplayer-Kompatibilität | mittel | Daten sind Teil der Mod → Spieler brauchen identische Mod-Version; ein **Server-/Shared-Plugin** verschärft das (Pflicht-Match). |
| **R10** | Linter nicht ausführbar | niedrig | Nur Windows; Ersatz: eigenes Python-Check-Skript. |
| **R12** | **Act-Auflösung per Id-Bereich / Id ≤ 255** (neu in Phase 1b, extern belegt, siehe Abschnitt 7) | mittel | Neue Gebiete nur in Act 5 und mit Id 138–255 anlegen, sonst Engine-Patch nötig (Town-Portal-Assertion ab 256, Act-Fehlzuordnung bei Act 1–4). |
| **R11** | Verwaiste Altlasten | niedrig | `pzboss1/pzend.ds1`, `labyrinth*.ds1` + 3 Presets, `lvlwarp.txt:91`, deaktivierte Cube-Rezepte. Nicht anfassen (Regel: nichts löschen), aber nicht versehentlich wiederverwenden. |

### Offene Fragen (bitte vor/zu Phase 2 beantworten)

> **Stand nach Phase 1b – Antworten des Nutzers:** (6) Push-Ziel: auf `claude/eloquent-ride-s83hfc` pushen; die Übernahme nach `feature/new-area-prototype` macht der Nutzer selbst. (1–5) Gebiets-Phase 2 ist **pausiert**, bis die Händler-Frage geklärt ist; falls sie doch kommt: wiederverwendete Layouts, Cube-Portal, englischer Text in allen Sprachen, Pitzilla/Labyrinth unberührt. (7) Plugin wird **nicht gestrichen, sondern auf den Händler umgewidmet** (Abschnitt 11). `MAP_FEASIBILITY.md` darf auf dem Claude-Branch committet werden (Ausnahme vom ursprünglichen „nicht committen“).
1. **Zugang:** Cube-Portal (empfohlen) oder fester Eingang (Stadt-Tor/Level-Ausgang)?
2. **Layout:** Reicht Wiederverwendung eines vorhandenen Basisspiel-DS1 / ein Maze-Level, oder willst du ein *eigenes* Layout (dann externer DS1-Editor, außerhalb dieser Sandbox)?
3. **Id-Strategie:** Zusammenhängend ab **138** (empfohlen; Lücken unbelegt) – und soll das Reimagined-Team (Discord `collin.h`, laut README) vorab einen Id-Block reservieren?
4. **Sprachen:** Englischer Text in allen 12 Fremdsprachen (wie im Labyrinth-Commit) oder echte Übersetzungen?
5. **Altlasten:** Pitzilla/Labyrinth reaktivieren, wiederverwenden oder unberührt lassen?
6. **Push-Ziel:** Diese Sitzung ist auf Branch `claude/eloquent-ride-s83hfc` festgelegt (Push nur dorthin ohne Freigabe); du forderst für Phase 2 den Branch `feature/new-area-prototype` von `next`. Bitte bestätigen, ob dorthin gepusht werden darf.
7. **Plugin:** Gibt es konkrete Zusatzfunktionen (Quest/Belohnung/UI), die ein Plugin leisten soll? Sonst Plugin-Teil streichen.

---

## 9. Empfohlener minimaler Prototyp (Phase 2, nach deiner Freigabe)

**Ziel:** Ein kleines, per Cube-Portal erreichbares Gebiet **Id 138** („Act 5 – Reimagined Testgrounds“), das **nur Basisspiel-Layouts wiederverwendet**, **keine bestehende Zeile ändert** und das Muster von Id 136/137 + Pitzilla kopiert.

**Variante A (Standard-Empfehlung): Preset-Klon** – am wenigsten Risiko, beweist die Pipeline.
**Variante B (optional, zweiter Schritt): Maze-Level** (`DrlgType=1`, zufälliger Dungeon aus vorhandenen Tiles, kein DS1) nach Muster Pitzilla 1.

### Schritte (Variante A) – jeweils in `excel/` **und** `excel/base/`, Zeilenenden je Datei erhalten
1. **`levels.txt` + `base/levels.txt`**: neue Zeile **am Ende** (nach Zeile 140; `levels.txt` CRLF, `base/` LF). Aus **Zeile 139 (Id 136)** klonen, per **Spaltenname** setzen: `Name = Act 5 - Reimagined Testgrounds`, `*StringName`, `Id = 138` (≤ 255 und Act 5, siehe R12), `Act = 4`, `OffsetX/Y = 5000/3000` (freie Stelle, nach Finale 5000/1000 und Colossal 5000/2000), `Vis/Warp = 0/-1`, `Waypoint = 255`, `LevelName/LevelWarp/LevelEntry` = neue Keys, `LevelGroup = Act 5 - Reimagined Testgrounds`, `PreventTownPortal = 1`. Monsterlisten aus Tristram (Zeile 40) übernehmen, `NumMon`/`MonLvl*` setzen, damit etwas spawnt.
2. **`lvlprest.txt` + `base/lvlprest.txt`**: neue Zeile **am Ende** (nach Zeile 1094): `Def = 1092`, `LevelId = 138`, `File1` = DS1-Pfad aus Zeile 1093 (`Act1/Tristram/Tri_Town4.ds1`) bzw. – falls ein „neutraleres“ Layout gewünscht – ein anderer in `lvlprest.txt` bereits referenzierter Basisspiel-Pfad; `Dt1Mask` von der Quellzeile kopieren.
3. **`levelgroups.txt` + `base/levelgroups.txt`**: neue Zeile am Ende (nach Zeile 102): Gruppe `Act 5 - Reimagined Testgrounds`, `CompleteThreshold=100`, `NameString` = neuer Key.
4. **`data/local/lng/strings/levels.json`**: 3 neue Einträge (`LevelName`, `LevelWarp`, `LevelEntry`) am Ende; Ids im Reimagined-Block ab **54000** (`fill-in-ids.js`), 13 Sprachen; Format wie `levels.json:7125-7175`. Muss `Json/KeyUsage` bestehen (Keys werden in `levels.txt`/`levelgroups.txt` referenziert).
5. **`cubemain.txt` + `base/cubemain.txt`** (CRLF in `excel/`): **ein** neues Rezept am Ende nach Muster `cubemain.txt:688`: `op=28`, billige Test-Inputs, Output **`Red Portal,lvl=138,qty=1`**.
6. **Automap, Objekte, Monster, Waypoint, Warps:** **keine Änderung nötig** (Automap über `LevelType`; Monster über vorhandene `monstats`; kein Waypoint, kein `lvlwarp`).
7. **Neues Skript** `scripts/check_levels.py` (nur *neue* Datei): Id-Lückenlosigkeit, Querverweise, Strings (13 Sprachen), Gleichheit der Bänke, Zeilenenden.
8. **Testanleitung** (`TESTING_NEW_AREA.md` o. ä.): `data/` per `copy-files.bat` ins `mods\Reimagined\Reimagined.mpq\data`; Spiel mit `-mod Reimagined -txt` **[ANNAHME zum Startparameter]**; neue Zeile aus `levels.txt` per Kommandozeile/Cube-Rezept erreichen; `lint/d2rlint.exe` auf Windows laufen lassen und gegen die Baseline vergleichen.

**Ausdrücklich nicht Teil des Prototyps:** Änderungen an `townwest.ds1`/`townwest.json`, neue DS1/HD-Presets, Waypoints, Plugin, Terror-Zonen-Aufnahme, Löschen/Umbenennen von Altlasten.
**Geplante Liste „bestehende Zeilen geändert“ (Phase 2): leer** – alle Änderungen sind Zeilen am Dateiende bzw. neue Dateien. Falls sich dies als falsch erweist, wird jede Ausnahme einzeln aufgelistet.

**Commits (Phase 2, klein & thematisch):** (1) Level-/Preset-/Gruppen-Zeilen, (2) Strings, (3) Cube-Portal, (4) Check-Skript, (5) Testanleitung.

---

## 10. Methodik / reproduzierbare Belege

- Spaltennamen/Ids/Zuordnungen: Python-`csv`-Auswertung der Tabellen (Tab-getrennt, BOM-sicher); Level-Ids 0–137, lückenlos; `DrlgType`-Zuordnung zu `lvlmaze`/`lvlprest` gezählt (70/70, 36/36).
- Historische Zustände: `git show 7f0ed3b3[^]:<pfad>`, `git log -S"…"` (Pickaxe) für „erstmals eingeführt“-Aussagen.
- SDK: alle Aussagen mit Datei(:Zeile) aus `../PluginSDK/README.md` und `include/D2RLPlugin/*.h`.
- Nicht geprüft: In-Game-Verhalten, Linter-Ausgabe, DS1-/DT1-Inhalte (Binärdateien), Blizzard-Basisdaten.


---

## 11. Fahrender Händler (Phase 1b – nur Analyse)

**Anforderung (Kurzform):** Ein Händler erscheint alle X Spielminuten in einer bestehenden Stadt, ist ansprechbar, öffnet ein Shop-Fenster;
Angebot pro Besuch zufällig (z. B. 3 normale, 3 Exceptional, 1–2 Elite-Uniques/Sets), **fester Preis in Runen**, nach X Minuten Wechsel,
Ankündigung im Chat, Bestand stabil pro Besuch (kein Reroll durch Verlassen/Neuladen).

Zusätzlich zu SDK und Mod-Daten wurden **zwei Community-Repos gelesen** (read-only geklont, Nachbarverzeichnisse):
`../RuffnecKk-D2RLoader-Suite` (@`6a2b369`, 2026-10-03; vom Nutzer genannt) und – weil die Suite es als Referenz zitiert (`research/references.json:46-52`) –
`../D2RL-Plugins` (eezstreet, offizielles Plugin-Pack, Shallow-Clone).

### 11.0 Kurzfazit

**Machbar – aber nicht als „echter NPC“, sondern als *virtueller Händler* (Panel + Plugin) auf der SDK-Ebene.** Ein echter, ansprechbarer NPC-Körper
mit nativem Shop ist mit der öffentlichen API **nicht** machbar und selbst mit nativen Hooks Neuland (keine dokumentierte Funktion zum Erzeugen von
Monstern/NPCs in beiden Referenz-Repos). Der Runen-Preis ist über Tabellen **unmöglich** (Währung ist fest Gold), über das SDK dagegen sauber lösbar
(Item-Transaktion: Runen verbrauchen → Item erzeugen). Eine Datenebenen-Notlösung (Cube-Rezepte) ist möglich, kennt aber weder Timer noch Zufall noch Bestand.

| Baustein | Urteil | Kern der Begründung |
|---|---|---|
| **Timer** (alle X Min.) | **machbar** | Kein Timer-Service im SDK, aber Overlay-Frame-Callback (`deltaTimeSeconds`, `overlay.h:63-70`) und Windows-Uhr genügen. „Spielminuten“ muss definiert werden (Echtzeit bei laufendem Spiel vs. Wanduhr). |
| **Spawn** eines NPC | **nicht machbar (jetzt)** | SDK: nur `EncounterService::runTestSpawn` (Boss-/Herald-Testspawns). Native: in `known-rvas.json` (1046 Einträge) existieren Item-, Portal-, Store- und NPC-Menü-Funktionen, **keine** zum Erzeugen von Monstern/NPCs. Eigener Reverse-Engineering-Aufwand, build-gebunden. Alternative: virtueller Händler ohne Körper. |
| **Chat-Ankündigung** | **eingeschränkt** | Kein Chat-Senden im SDK. Möglich: lokale Konsole (`WriteConsoleMessage`) und Overlay-Text/-Toast; im Mehrspielerspiel Host→Clients per `NetworkService`-Nachricht, die jeder Client lokal anzeigt. |
| **Shop-UI** | **machbar (eigenes Panel), nicht über den nativen Vendor** | Nativer Vendor-Panel ist per SDK weder abfangbar noch ersetzbar (`item_interactions.h:67-71`: Vendor-Interaktionen werden nicht gemeldet). Eigenes Panel + Item-Grid + Tooltip-Listener + Item-Interaction-Listener sind SDK-Funktionen. |
| **Runen-Preis** | **machbar (Plugin) / unmöglich (Tabellen)** | Tabellen: nur Gold (`npc.txt`, `cost`). Plugin: `ItemService::executeTransaction` verbraucht Runen und erzeugt das Item atomar (`item.h:263-290`). |
| **Bestand / Persistenz** | **machbar** | Items in einer *Custom Player Page* werden mit dem Charakter gespeichert (`inventory.h:125-127`); zusätzlich lässt sich das Zeitfenster deterministisch aus der Uhr ableiten (kein gespeicherter Zustand nötig). Per-Charakter-Zustand gibt es sonst nicht im SDK. |
| **Elite-Uniques/Sets im Angebot** | **machbar (Plugin) / unmöglich (Vendor-Tabellen)** | `createItem` mit `Quality::Unique/Set` + `qualityRecordId` (`item.h:171-203`); Vendor-Tabellen kennen nur normal/magic. |
| **Mehrspieler** | **eingeschränkt** | Item-Änderungen nur lokal/auf dem TCP/IP-Host; Clients melden per Plugin-Nachricht (`README.md:720`); nicht auf Battle.net (`README.md:1015`). |

### 11.1 Plugin-SDK (0.3.0, ABI 4): Matrix a–g

Legende: **vorhanden** = Funktion in öffentlichem Header · **nur lesend** · **nicht vorhanden** (Header-Suche nach `timer|tick|chat|npc|vendor|shop|merchant|gold|spawn|persist|serial|dialog|talk|clock|gameTime` ergab nur die unten genannten Treffer).

| # | Fähigkeit | Status | Beleg (SDK) |
|---|---|---|---|
| a1 | **Timer/Tick-Callbacks** | **nicht vorhanden** als Service. Ersatz: `OverlayService` ruft pro Frame `FrameCallback` (UI-Thread, `frameNumber`, `deltaTimeSeconds`) – **[ANNAHME]** nur solange der Loader das Overlay zeichnet (`README.md:472-475`: „The loader decides when drawing runs“). `ThreadService::runOnGameThread` stellt **einmalige** Callbacks „in ein späteres autoritatives Game-Update“ ein (erneutes Einreihen = Tick; nicht auf TCP/IP-Clients). | `overlay.h:63-70,95`, `threads.h:26-31,44` |
| a2 | **Zugriff auf Spielzeit** | **nicht vorhanden** (kein Frame-/Game-Tick-Zähler, keine Tageszeit). Nur Lifecycle-Ereignisse (GameJoined/GameLeft/LocalPlayerReady/ActChanged/LevelChanged) und Echtzeit über die Windows-API im Plugin selbst. | `lifecycle_events.h:30-45` |
| b | **NPC/Monster spawnen & entfernen** | **nicht vorhanden.** Nur `EncounterService`: Herald-/Boss-*Entscheidungen* (`registerProvider`) und `runTestSpawn` (Testspawns `HeraldReplacement/BossClones/HeraldBossGroup`, braucht rohen `D2R::Game::Client*`); kein Entfernen. | `encounters.h:124-168` |
| c | **Chat-/Systemnachrichten senden** | **nicht vorhanden** (kein Chat). **Vorhanden:** lokale Konsole `WriteConsoleMessage(kind)` (Output/Debug/Warning/Error) und `ExecuteConsoleCommand` (Core-Export 14, schaltet kurz das Cheat-Flag), Overlay-Text (`TextRequest`, ≤ 4096 B), `NetworkService`-Nachrichten (≤ 224 B, Host→Client, nicht Battle.net). | `console.h:22,110`, `core_exports.h:36,177`, `overlay.h:136`, `network.h`, `README.md:1013-1015` |
| d | **Dialog mit NPC abfangen / Shop-UI öffnen oder ersetzen** | **nicht vorhanden** für native NPC/Vendor. `ItemInteractionService` meldet ausdrücklich **keine** Vendor-/Trade-/Corpse-Interaktionen. `SharedEventService::registerUiMessageListener` sieht *UI-Nachrichten* (Target/Command/Text) und darf sie „konsumieren“ – ob ein Vendor-Kauf eine solche Nachricht ist, ist nicht belegt (**[ANNAHME]**: eher nicht; belegt ist nur `VendorPanelMessage:RefreshAll`, siehe 11.4). **Vorhanden:** eigene Panels (`PanelService`, Layout-JSON, `GameplayLeftSlot` = Plugin-Panel links + Inventar rechts), `WidgetService` (finden, sichtbar/aktiv schalten, UI-Action senden; **kein `setText`**), Controls-Aktionen (Hotkey), Konsolenbefehle. | `item_interactions.h:67-71`, `shared_events.h:124-130`, `panels.h:52-53,131-140`, `widgets.h`, `input.h` |
| e1 | **Inventar/Items erzeugen** | **vorhanden** – `createItem` (Code, Qualität, Set/Unique-Zeile, Affix-Ids, Item-Level, Sockets, Ethereal, Eigenschaften) in Inventar/Cube/Stash/Custom-Page/Cursor/**Boden**; Unique-„einmal pro Spiel“-Regel per Flag `AllowDuplicateUnique` aufhebbar. Nur lokal bzw. TCP/IP-Host (`NotAuthoritative` auf Clients). | `item.h:61-76,86,171-203,476-478`, `README.md:716-766` |
| e2 | **Items im „Shop“ anlegen** | **vorhanden als Custom Player Page** (1–16 × 1–16 Zellen, an ein `InventoryGridWidget` des eigenen Panels bindbar; Item-Policy bestimmt, was hinein darf). Kein Eingriff in nativen Vendor-Bestand. | `inventory.h:129-163,196-202`, `panels.h` (`bindPlayerPageGrid`), `README.md:679-715` |
| e3 | **Gold- vs. Rune-Abzug** | **vorhanden als Tausch-Primitiv:** `executeTransaction` (bis 64 Eingänge/Ausgänge; Eingänge ganze Items oder Stack-Teile, Ausgänge neu erzeugt; alles-oder-nichts) bzw. `executeExistingItemTransaction` (Debit/Edit/Move bis 4096 Operationen). **Nicht vorhanden:** Umbiegen der *nativen* Gold-Abbuchung des Vendors. | `item.h:17-19,244-290,311-337` |
| f | **Persistenz (Speichern/Laden)** | **teilweise:** Items auf Custom Pages werden von D2RLoader beim Laden wiederhergestellt („save/load protected“); Plugin-Konfig `ReadConfig/WriteConfig` ist **eine globale TOML pro Plugin**, nicht pro Charakter; Plugin-Historie im Charakter ist nur Metadatum. **Nicht vorhanden:** Key-Value-Speicher pro Charakter. | `inventory.h:125-127`, `context.h` (`ReadConfig/WriteConfig`), `README.md:92-98,193-196` |
| g | **UI-Overlay / Controls** | **vorhanden:** `OverlayService` (Linien, Rechtecke, Text, Clip; **nur Anzeige**, kein Input), `InputService` (benannte Aktionen im Controls-Menü, nur Tastatur laut Suite-README), Panels/Widgets, Tooltip-Erweiterung (`SharedEvents::ItemTooltipListener`: eigene Zeilen in Item-Tooltips, z. B. „Preis: 2× Ith“). | `overlay.h`, `input.h`, `shared_events.h:111-123`, Suite-`README.md:119-120` |

**Folgerung:** Ein vollständiger Shop lässt sich **ausschließlich mit SDK-Services** bauen (Panel + Custom-Page-Grid + Tooltip-Preis + Item-Interaction + Item-Transaktion + Overlay-Toast + Timer über Frame-Callback). Das ist der einzige Pfad, der **nicht** build-spezifisch ist (Services werden per Version/Größe verhandelt, `README.md:276-286`).

### 11.2 Datenseite im Mod: wie Shops definiert sind

[BELEGT im Repo]
- **`data/global/excel/npc.txt`** (1 288 B, 17 Zeilen: `gheed, charsi, akara, lysander, drognan, elzix, fara, hratli, alkor, ormus, asheara, jamella, halbu, malah, drehya, larzuk, nihlathak`): Spalten `buy mult`, `sell mult`, `rep mult`, Quest-Varianten A–C, **`max buy` (N/H)**. **Keine Währungsspalte** – Preise sind Gold.
- **Sortiment steckt in den Item-Tabellen:** `armor.txt` (Spalten 75–159), `weapons.txt` (77–161), `misc.txt` (77–161): je Händler `<Npc>Min`, `<Npc>Max`, `<Npc>MagicMin`, `<Npc>MagicMax`, `<Npc>MagicLvl` (z. B. `CharsiMin` … `AnyaMagicLvl`). Preis = `cost` (+ `gamble cost`, `multibuy` in `misc.txt:170`).
  → **Genau diese 17 Händler können Stock haben; die Spalten sind fest verdrahtet.** Ein *neuer* Händler-NPC kann per Tabelle **keinen eigenen** Stock bekommen (nur einen vorhandenen mitbenutzen) **[ANNAHME zu neuen NPC-Ids; belegt ist nur das 17er-Schema, auch in `D2RL-Plugins/.../plugin-shared.h:208-213` (`nVendorMin[17]` usw.)]**.
- **Vendor-Qualität:** Stock ist normal/magic (offizielles Plugin kann optional „Rare“ zuschalten, `D2RPlugins.json` `vendorOverhaul.randomRareItems`). **Unique/Set im Vendor-Stock existieren nicht** – dafür bräuchte man Gamble (`gamble.txt`, zufällig, Gold) oder Code.
- **Währung:** `cost` in Gold; Runen sind in `misc.txt` nicht stapelbar (`stackable=0`, `misc.txt:105,110,137`) und haben Gold-Werte (El 560, Ith 2240, Zod 134 540) – ein „Rune-Preis“ per Tabelle ist nicht darstellbar.
- **NPC-Definition:** `monstats.txt` (78 Zeilen mit `npc=1` oder `inventory=1`); relevante Spalten `AI` (z. B. `Npc`), `npc` (79), `interact` (80), `inventory` (81), `inTown` (82). Strings: `NameStr` → `monsters.json`. **Dialoge/Menüeinträge sind nicht in Tabellen**, sondern in einer nativen Aktions-Registry (72 Byte/NPC; Suite-Research `known-rvas.json: CLIENT_BuildNpcInteractionMenu`).
- **Platzierung/Zeitsteuerung:** `monpreset.txt` bildet DS1-Platzhalter auf NPCs ab (`Act`, `Place`, `DS1 ID#`; z. B. Act 1: `gheed, cain1, akara, kashya, warriv1, charsi …`) – **statisch**. `actinfo.txt:1` kennt `wanderingnpcstart/-range` (Act 1: 0/1, übrige 0/0); die zugehörige `WanderingMon`-Tabelle liegt **nicht** im Repo (Basisspiel) → nicht analysierbar. **Keine** datengetriebene Möglichkeit für „erscheint alle X Minuten“.

Wiederverwendbare Stadt-NPCs (aus `monstats.txt` `npc=1`, `inTown=1`, abgeglichen mit `npc.txt`) – **als Körper/Name, nicht als Shop-Logik**.
Achtung: `inventory=1` in `monstats.txt` heißt **nicht** „hat Shop“ (u. a. `warriv1/2`, `atma`, `meshif1/2`, `greiz` tragen es, stehen aber nicht in `npc.txt`); **echte Händler = die 17 Namen in `npc.txt`**.

| Act | Händler (in `npc.txt` → Stock-Spalten vorhanden) | NPCs ohne Stock (Dienste/Dialog/Deko) |
|---|---|---|
| 1 | gheed, charsi, akara | warriv1 (Reise), cain1, kashya, rogue1/3 (Dummy) |
| 2 | lysander, drognan, elzix, fara | warriv2, atma, meshif1, greiz, geglash, jerhyn, cain2, act2male/-female/-child, act2vendor1/2 (`AI=Vendor`, ohne Stock) |
| 3 | hratli, alkor, ormus, asheara | meshif2, cain3, natalya, act3male/-female |
| 4 | jamella, halbu | malachai, tyrael2, cain4 |
| 5 | malah, drehya, larzuk, nihlathak | qual-kehk, cain6, tyrael3 |

Vorschläge: Für ein *thematisches* Händler-Modell eignen sich NPC-Rows mit `AI=Towner`/`Vendor` (z. B. `act2vendor1/2`) – sie sind in `monpreset.txt` bereits platziert (Act 2) und tragen **keinen** nativen Shop; ihr Name kann über `monsters.json` geändert werden. **[ANNAHME]**: ob ein solcher NPC ansprechbar gemacht werden kann, ohne Hook, ist nicht belegt (`interact` ist bei `act2vendor1/2` leer).

### 11.3 Fallback ohne Plugin: Horadric Cube als „Händlertisch“

[BELEGT] `cubemain.txt`: 15 900 Rezepte, 106 Spalten. Header: `description, enabled, firstLadderSeason, lastLadderSeason, min diff, version, op, param, value, class, numinputs, input 1 … input 7, output, lvl, plvl, ilvl, mod 1–5 (chance/param/min/max), output b, output c …, *eol`.
- **Verwendete `op`-Werte:** 18 (14 396×), 16 (341×), 15 (207×), 17 (19×), **28 „Special“ (7×)**, leer (930×). Native, in Reimagined **ungenutzte** Bedingungen laut Linter-Config (`lint/config.json`, `cubeOps`): `1` Tag-des-Monats-Fenster, `2` Wochentag, `3–14` Spieler-Stats, `15–27` Stats/Mod-Klasse des ersten Inputs.
- **Rune-Rezepte:** 3 890 Rezepte mit Runen-Inputs (ohne „CORRUPT…“), z. B. `cubemain.txt:290` `amu,mag + r06(Ith) + r01(El) → usetype,mag`; **feste Unique-Ausgabe per Namen:** `cubemain.txt:303` `rin,mag + jew,rar + r07 + r08 + r09 → Barilzar's Mazed Band`.
- **Portal-Rezept (Colossal Summit):** `cubemain.txt:688` – `op=28`, `numinputs=5`, Inputs `ua1…ua5`, Output `Red Portal,lvl=137,qty=1`, `min diff=2`; (Pitzilla-Altlasten `:689-690`, `enabled=0`).

| Was mit Cube-Rezepten geht | Was nicht geht |
|---|---|
| **Fester Rune-Preis:** `r0x + r0y (+ Token) → festes Zielitem` (Unique/Set per Namen, Exceptional/Elite-Basis per Code), unbegrenzt oft, überall mit Cube | **Kein Timer**, kein Erscheinen/Verschwinden, **keine Ankündigung** |
| **Zugangsbeschränkung** über Spieler-Stats (`op 3–14`, z. B. Level/Klasse) oder **Realkalender** (`op 1/2`: Tag/Wochentag) → grobe „Händler-Tage“ (aus Systemdatum, **manipulierbar** – [ANNAHME]) | **Kein Zufallsangebot pro Besuch** – Ergebnis ist je Rezept deterministisch (Zufall nur *innerhalb* eines Ausgabeitems, z. B. Affixe, `usetype`) **[ANNAHME zu „erstes passendes Rezept gewinnt“]** |
| Rezepte als Datenänderung (`cubemain.txt` + `base/cubemain.txt`, CRLF/LF beachten) – **keine Plugin-Pflicht, kein Hooking** | **Kein Bestand** (nie ausverkauft), keine Händler-Optik/-Fenster |
| Katalog „Händlertisch“ = Menge fester Rezepte (Obergrenze nur durch Tabellengröße) | Preis in *Runen-Anzahl* nur über mehrere Input-Slots (max. 7 Inputs); nicht stapelbare Runen → N Einzelitems |

Fazit Cube: Taugt als **dauerhafter Rune-Shop ohne Zufall/Timer** (Stufe „C0“), nicht als Händler-Erlebnis.

### 11.4 Referenz: was die Community-Suite tut (und was daraus für Spawn/Shop folgt)

**Stand:** RuffnecKk-Suite 1.4.2 (2026-09-20, Quellstand 2026-10-03), 22 Plugins + 18 Memory-Patches; Voraussetzungen **D2R 3.3.93847, 3.2.92777 oder Steam 3.3.93787** und **D2RLoader 1.3** (`README.md:21-25`). Das passt zur Reimagined-Datenbasis: `data/global/dataversionbuild.txt` = `93847` **[BELEGT, Querbezug]**. Installation global (`<D2R>/d2rloader/`) **oder** mod-spezifisch (`<D2R>/mods/<mod>/d2rloader/`), „nie beides gleichzeitig“.

**Wie sie Items bewegen/ändern – SDK-API vs. Memory-Patch** (Auszählung über `plugins/*/src/*.cpp`, 27 Plugin-Ordner):
- **Alle 27** setzen mindestens einen **Inline-Hook auf feste RVAs mit Erwartungsbytes** ein (Fingerprint-Prüfung, bei Abweichung wird das Plugin verweigert); **nur `mass-identify`** nutzt `ItemService`/`InventoryService` (13/15 Verweise).
- **Bulk Currency Deposit** („Automatic Materials Deposit“): SDK-Panel/Input/Thread **plus** 2 Hooks und rund 58 RVA-Verweise im Quelltext; Items wandern über die *native* Advanced-Stash-Routine (`TransferItemToInventoryPage` `0x15F8B0`, `GetAdvancedStashDestination` `0x46DA50` …), nicht über `ItemService.executeExistingItemTransaction`.
- **Potion Auto Pickup / Cube Quick Move / Equipped Item to Cube:** je 1–5 Inline-Hooks auf native Inventar-/Pickup-Funktionen.
- **Vendor Stock Refresh:** 4 Inline-Hooks (`ConfigureVendorInteraction 0x502F60`, `EntityAction 0x4B0470`, `ConfigureVendorPanel 0x2411E0`, `SendVendorRefresh 0x10F520`) – baut einen Reroll-Button in den nativen Vendor-Panel und sendet Paket-Opcode `0x38` (9 Byte `{opcode, action, npcGuid}`).
- Wichtiger Kontext: Die Suite **vendort die SDK-Versionen v3 (`4933e2c`) und v4 (`6eb8f8b`)** (`research/d2rloader-1.3-public-audit.md`). In diesen Ständen gab es `ItemService` (Id 15, schon in v3) und `ItemInteraction` (v4) bereits; **neuer** sind `Encounter` (2026-09-20), `Mutation`, `Overlay`, `PluginCommunication` (2026-09-22, SDK-Commits `46ba079`/`4aec92a`) sowie die Erweiterungen `splitStack`/Shared-Stash-Writes/`augmentItemAffix`. Die Hook-Lastigkeit der Suite spiegelt also **überwiegend echte API-Lücken** (Pickup, Vendor, Schaden, Karten) und nur zum Teil SDK-Alter – sie ist **kein** Beleg, dass `ItemService`-basierte Tauschlogik nicht trägt; sie ist aber ein Beleg, dass diese Tauschlogik in der Suite **kaum praktisch erprobt** ist (nur `mass-identify`).

**Was sich daraus für Spawn/Shop ableiten lässt**
1. **Nativer Vendor ist serverseitig und dokumentiert**: Pro NPC ein `VendorChainEntry` (0x78 B, gefüllt-Flag `+0x34`, Refresh-Flag `+0x35`, Zeitstempel `+0x38`); `D2GAME_NPC_FillStoreInventory 0x53C9F0`, `GenerateStoreItem 0x540EA0`, `ClearVendorInventory 0x502F00`. **Der native Stock hat bereits einen Timer:** `UpdateVendorRefreshState 0x503290` setzt per `GetTickCount64` nach 240 000 ms (4 min) das Refresh-Flag (`known-rvas.json`; Quelle: Research, Build 92777). Das offizielle Pack (`D2RL-Plugins/src/plugin-items/items-main.cpp:287-376`) **schreibt `FillStoreInventory` in C++ neu** (Vendor-Overhaul) – ein Plugin kann also Stock-Erzeugung ersetzen, aber nur für die 17 festen Vendor-Slots und mit den nativen Qualitäten.
2. **Gold-Abzug ist eine einzelne Server-Funktion:** `D2GAME_NPC_TryDeductGold 0x5416D0 (game, player, amount)`. Sie auf Runen umzubiegen wäre ein Hook – **aber** die **Client-UI zeigt weiter Gold-Preise**; ein sauberer Rune-Preis im nativen Vendor-Fenster erfordert Client- *und* Server-Hooks plus Preisanzeige (hoch riskant, build-gebunden). Das SDK-Transaktionsmodell (11.1 e3) ist klar überlegen.
3. **NPC-Menü ist hookbar, aber nativ:** Client baut das Menü aus einer 72-Byte-Aktions-Registry (`CLIENT_BuildNpcInteractionMenu 0x1147A0`, `UI_AddNpcMenuEntry 0x1E9030`); Server-Callbacks `0x34/0x38/0x39` sind autoritativ. Einen eigenen Menüeintrag („Händler: Angebot ansehen“) zu ergänzen ist denkbar, bleibt aber Native-Hook-Arbeit.
4. **Spawn:** Aus beiden Repos ableitbar sind nur **Item-Erzeugung** (`ITEMS_CreateItemUnit 0x43D530`), **Portal-Objekte** (`D2GAME_CreatePortalObject 0x432CE0`) und die interne `spawn`-Konsole für *Items* (`research/cheat-spawn.md`: `spawn <code>,qty=…,typ=u,lvl=…,inv=1`, Boden/Inventar). **Kein Monster-/NPC-Spawn.** Das ist die größte Lücke für einen „echten“ Händler.
5. **Persistenz-Muster der Suite:** nur Plugin-TOML (`WriteConfig` in `floating-damage`, `mapsense`), kein Per-Charakter-Speicher; das große Persistenz-Plugin ISC12 (rund 578 RVA-Verweise, 11 Patch-Aufrufe) erweitert das Speicherformat und ist als *Risikobeispiel* zu lesen („Requires ISC12-compatible save files“).
6. **Reife-Hinweis der Autoren:** „current source changes are not a new Suite release or a claim that every component has completed runtime testing“ (`README.md`); Audit: „Gameplay, save/reload and multiplayer remain untested“ für Loader 1.3 (`research/d2rloader-1.3-public-audit.md`). **Alles Native ist hier build-gepinnt und nur punktuell runtime-getestet.**

### 11.5 Risiken (Händler)

| # | Risiko | Einstufung | Hinweis |
|---|---|---|---|
| H1 | **Spiel-Updates** (Blizzard-Patches ändern RVAs/Layouts) | SDK-Services: niedrig–mittel; Native Hooks: **hoch** | Services werden pro Version/Größe verhandelt (`README.md:276-286`) und Tabellenzeilen „für den laufenden Build“ gelesen (`README.md:1319`); Hooks brauchen Erwartungsbytes pro Build, die Suite pflegt 3 Builds. Reimagined-Daten sind an Build 93847 gebunden. |
| H2 | **Build-spezifische Adressen** | hoch (nur Variante „echter NPC“) | siehe 11.4; keine Offsets im SDK. |
| H3 | **Multiplayer/LAN** | mittel | Shared/Server-Plugins müssen zwischen Host und Clients übereinstimmen (`README.md:193-196`); Item-Änderungen nur Host/Lokal; `NetworkService` nur Plugin-Nachrichten ≤ 224 B; **nicht Battle.net**. Händler-Zustand muss vom Host kommen (Determinismus aus gemeinsamem Seed). |
| H4 | **Kompatibilität mit Reimagined-Updates** | mittel | Unique/Set-Auswahl über `qualityRecordId` = *nullbasierte Zeile* in `uniqueitems.txt`/`setitems.txt` – **bricht, wenn Reimagined Zeilen einfügt/umsortiert**; besser Namen→Zeile beim Laden über `DataTableService` auflösen. Runen-/Item-Codes ändern sich selten, Preislogik gehört in eine plugin-eigene Tabelle. |
| H5 | **Plugin-Zwang für Spieler** | mittel | Zusätzlicher Download (D2RLoader 1.3 + Plugin-DLL); mod-spezifischer Pfad `<D2R>/mods/Reimagined/d2rloader/plugins/` möglich; Reimagined-Launcher kennt das (noch) nicht **[ANNAHME]**. Koexistenz mit anderen Loader-Plugins (Suite-Changelog erwähnt Überschneidungen mit „CelestialRay's Suite“). |
| H6 | **Exploits** | mittel | Items auf Custom Page dürfen nicht kostenlos entnehmbar sein (Item-Policy/`Consume` in `ItemInteractionListener`; Drag-&-Drop-Verhalten **nicht verifiziert**); Stock beim Weggang sauber entfernen (`destroyItem`), auch bei Spielabbruch; Wanduhr-Variante ist per Systemzeit manipulierbar; Unique-„einmal pro Spiel“-Regel. |
| H7 | **Spiel-/Wirtschaftsbalance** | mittel | Runenpreise und Elite-Uniques/Sets im Handel verändern die Reimagined-Ökonomie (außerhalb dieser Analyse). |
| H8 | **Nicht testbar in dieser Sandbox** | – | Linux, kein D2R, kein MSVC/Windows – alle Aussagen sind Quell-/Dokumentationsbefunde. |

### 11.6 Architekturvarianten

| Variante | Beschreibung | Aufwand | Risiko | Erfüllt Anforderung |
|---|---|---|---|---|
| **A – Virtueller Händler (SDK-only)** | Plugin: Timer + Overlay-Toast/Konsole („Der Händler ist da“), Panel (`GameplayLeftSlot`) mit Custom-Page-Grid, Stock per `createItem`, Preis im Tooltip, Kauf per `executeTransaction`/Item-Interaction | mittel | niedrig–mittel | alles **außer** „ansprechbarer NPC-Körper“ und „Chat“ (stattdessen Overlay/Konsole) |
| **B – Cube-Händlertisch (Daten)** | Feste Rune-Rezepte in `cubemain.txt` (optional Kalender-`op`) | gering | niedrig | nur „fester Rune-Preis“; kein Timer/Zufall/Bestand |
| **C – Echter NPC (Native)** | Hooks für NPC-Spawn, NPC-Menü, ggf. Vendor-Fill/Gold-Abzug | **sehr hoch** (Reverse-Engineering, kein Spawn-RVA bekannt) | **hoch** (build-gebunden) | vollständig – aber unbelegt machbar |
| **D – Vorhandener NPC + Tabellenstock** | Vorhandenen Vendor (z. B. Halbu/Jamella) per `<Npc>Min/Max` umkonfigurieren | gering | niedrig | nur Gold, normal/magic, kein Timer – **keine** der Kernanforderungen |

### 11.7 Empfohlener Stufenplan (kleinster testbarer Schritt zuerst)

0. **Spike (1 Plugin, kein Gameplay):** Minimal-Plugin auf SDK 0.3.0 (Rolle `Client`/`Shared` klären), geladen aus `<D2R>/mods/Reimagined/d2rloader/plugins/`. Registriert Konsolenbefehl `merchant-status`, zählt über den Overlay-Frame-Callback Echtzeit und zeigt alle 60 s einen Overlay-Toast + Konsolenzeile. **Prüft:** Plugin-Laden in der Reimagined-Umgebung, Timer, Ankündigung, Verhalten bei Pause/Menü/Spielwechsel. *(Braucht Windows/MSVC; hier nicht baubar.)*
1. **Statischer Shop:** Panel + Layout-JSON + Custom-Page-Grid; 3 feste Items per `createItem`; Preis als Tooltip-Zeile; Kauf = `executeTransaction` (n Runen → Item). Prüft Panel-Bedienung, Item-Interaction, Atomarität, „nicht kostenlos entnehmbar“.
2. **Zufallsangebot + Rotation:** Angebot (3 normal / 3 Exceptional / 1–2 Elite-Unique/Set) deterministisch aus `Seed = f(Charakter, floor(Zeit/X))` → stabil bei Reload; Rotation, Ankündigung, Aufräumen beim Weggang; Namen→Zeile-Auflösung für Uniques/Sets (H4). Preis als plugin-eigene Custom-Table (`CustomTableService`, `d2rloader/<plugin-id>/…`).
3. **Mehrspieler/TCP-IP:** Host-autoritativ (`NetworkService`), Clients nur UI; Matching-Konfiguration (`d2rl.match`).
4. **Optional/gesondert entscheiden:** echter NPC (Variante C) erst nach einem **separaten Reverse-Engineering-Spike** (gibt es einen verwendbaren Monster-Spawn-Pfad? NPC-Menü-Eintrag?). Bis dahin ist der „Händler“ ein Panel mit Hotkey/Button und optionalem Mini-HUD.
- **Parallel, unabhängig vom Plugin:** Stufe **C0** = Cube-Händlertisch (Variante B) als sofort liefernder Daten-Fallback.

### 11.8 Offene Fragen an dich (Händler)

1. **Sichtbarer NPC Pflicht?** Reicht ein „Händler ist da“-Panel (Hotkey/Button + Toast), oder muss der Händler als Figur in der Stadt stehen? *(Schlüsselfrage: A vs. C.)*
2. **„Spielminuten“:** Echtzeit, solange das Spiel läuft (Pause/Menü ausnehmen?) oder Wanduhr (stateless, aber per Systemzeit manipulierbar)?
3. **Mehrspieler:** nur Einzelspieler, oder auch TCP/IP/LAN? (Battle.net ist ausgeschlossen.)
4. **Plugin-Zwang akzeptabel?** Spieler brauchen D2RLoader 1.3 + DLL (Windows). Soll das Reimagined-Team/Launcher das mittragen, oder ist ein reiner Daten-Fallback (B) vorzuziehen?
5. **Preise/Währung:** Welche Runen, welche Anzahl je Item-Klasse? Runen sind nicht stapelbar (N Einzelitems werden verbraucht, max. 64 Eingänge je Transaktion).
6. **Angebotsquellen:** Welche Basis-Items (Exceptional/Elite) und welche Unique-/Set-Pools? Duplikate (`AllowDuplicateUnique`) erlaubt?
7. **Stadt/Act:** Eine feste Stadt oder je Act einen Händler? (Bei Variante A irrelevant; bei C entscheidend.)
8. **Pflege:** Soll das Plugin ein eigenes Repo/Release neben `d2r-reimagined-mod` werden (empfohlen: SDK-only, keine RVAs), und wer pflegt es bei Reimagined-/Blizzard-Updates?

### 11.9 Quellen und Grenzen (Abschnitt 11)

- Repo: `data/global/excel/{npc,monstats,monpreset,actinfo,cubemain,misc,armor,weapons,gamble,storepage}.txt`, `lint/config.json`.
- SDK: `PluginSDK/include/D2RLPlugin/{threads,overlay,console,core_exports,encounters,item,inventory,panels,widgets,shared_events,item_interactions,network,lifecycle_events}.h`, `README.md`.
- Suite: `RuffnecKk-D2RLoader-Suite/{README.md,CHANGELOG.md,plugins/*,research/d2r/known-rvas.json,research/d2r/findings.md,research/cheat-spawn.md,research/d2rloader-1.3-public-audit.md}`; Offiziell: `D2RL-Plugins/{README.md,D2RPlugins.json,src/plugin-items/*,src/plugin-shared/include/plugin-shared.h}`.
- **Nicht geprüft:** Lauffähigkeit irgendeines Plugins, Verhalten des nativen Vendors im Spiel, Drag-&-Drop-Verhalten auf Custom Pages, Kompatibilität mit den Reimagined-Layouts (Inventar-Layouts im Repo wurden bereits für ein Loader-Plugin angepasst, `86aae80e`), alle RVA-Angaben stammen aus fremder Research für Build 92777/93847 und sind **nicht selbst verifiziert**.
