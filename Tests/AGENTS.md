# Zielbild

* Eingabe: Ordner `tests_raw/` mit deinen **.docx**-Dateien (z. B. aus deiner ZIP: `LZK Überstiegstest.zip` → entpacken).
* Ausgabe: `data/db.sqlite3` mit:

  * `category` (5 Kategorien à 20 Punkte, fix)
  * `rule_global` (deine komplette Regelspezifikation, maschinenlesbar)
  * `provenance` (Datei-Hash/Größe/Name)
  * `source_block` (alle Paragraphen/Tabellen der DOCX als Rohtext-Blöcke)
  * `asset` (alle eingebetteten Bilder mit SHA-256, Pfad, Größe)
  * (optional) `template`, `template_var`, `template_rule` als **Skeleton** (ohne Generierung)

---

## 0) Voraussetzungen

* OS: macOS, Linux oder Windows
* Tools: Git, Python ≥ 3.10, `sqlite3` CLI
* Python-Pakete: `python-docx`, `pillow`, `tqdm`, `tabulate`, `pyyaml` (plus ggf. `sqlalchemy`, `pydantic` für spätere Ausbaustufen)
* Optional: **Git LFS** (für große Bild-Assets)

---

## 1) Projekt anlegen

```bash
mkdir ueberstieg-db && cd ueberstieg-db
mkdir -p data/assets data/rules data/manifests src tests tests_raw
printf "# Überstiegstest DB\n" > README.md
```

Falls du eine ZIP hast:

```bash
unzip "LZK Überstiegstest.zip" -d tests_raw
```

---

## 2) Virtuelle Umgebung & Dependencies

```bash
python -m venv .venv
# macOS/Linux:
source .venv/bin/activate
# Windows PowerShell:
# .venv\Scripts\Activate.ps1

pip install --upgrade pip
pip install python-docx pillow tqdm tabulate pyyaml
```

*(Optional für später:)*
`pip install sqlalchemy pydantic`

---

## 3) SQLite-Schema erstellen

Datei `data/schema.sql`:

```sql
PRAGMA foreign_keys=ON;

CREATE TABLE IF NOT EXISTS category (
  id INTEGER PRIMARY KEY,
  key TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  points_total INTEGER NOT NULL CHECK(points_total = 20),
  sort_index INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS template (
  id INTEGER PRIMARY KEY,
  category_id INTEGER NOT NULL REFERENCES category(id),
  code TEXT UNIQUE NOT NULL,
  title TEXT NOT NULL,
  expr TEXT NOT NULL,
  difficulty INTEGER NOT NULL CHECK(difficulty BETWEEN 1 AND 5),
  result_min REAL,
  result_max REAL,
  bracket_depth INTEGER DEFAULT 0,
  notes TEXT
);

CREATE TABLE IF NOT EXISTS template_var (
  id INTEGER PRIMARY KEY,
  template_id INTEGER NOT NULL REFERENCES template(id) ON DELETE CASCADE,
  name TEXT NOT NULL,
  domain_type TEXT NOT NULL CHECK(domain_type IN ('RANGE','SET','CONST','FRACTION')),
  domain_json TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS template_rule (
  id INTEGER PRIMARY KEY,
  template_id INTEGER NOT NULL REFERENCES template(id) ON DELETE CASCADE,
  rule_key TEXT NOT NULL,
  params_json TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS asset (
  id INTEGER PRIMARY KEY,
  kind TEXT NOT NULL,                 -- image | diagram | other
  rel_path TEXT NOT NULL,             -- data/assets/<doc>/<file>
  sha256 TEXT NOT NULL,
  width_px INTEGER, height_px INTEGER,
  alt TEXT
);
CREATE UNIQUE INDEX IF NOT EXISTS ux_asset_path ON asset(rel_path);
CREATE INDEX IF NOT EXISTS ix_asset_sha ON asset(sha256);

CREATE TABLE IF NOT EXISTS template_asset (
  template_id INTEGER NOT NULL REFERENCES template(id) ON DELETE CASCADE,
  asset_id INTEGER NOT NULL REFERENCES asset(id) ON DELETE CASCADE,
  PRIMARY KEY (template_id, asset_id)
);

CREATE TABLE IF NOT EXISTS source_block (
  id INTEGER PRIMARY KEY,
  source_doc TEXT NOT NULL,           -- Dateiname.docx
  block_index INTEGER NOT NULL,
  kind TEXT NOT NULL CHECK(kind IN ('paragraph','table','header','footer')),
  text TEXT,
  asset_hint TEXT
);

CREATE TABLE IF NOT EXISTS provenance (
  id INTEGER PRIMARY KEY,
  source_doc TEXT NOT NULL,
  sha256 TEXT NOT NULL,
  bytes INTEGER NOT NULL,
  imported_at TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS rule_global (
  id INTEGER PRIMARY KEY,
  section TEXT NOT NULL,              -- z.B. "fundamental", "grundrechenarten"
  key TEXT NOT NULL,                  -- z.B. "total_points", "tasks"
  value_json TEXT NOT NULL            -- JSON-String
);

CREATE TABLE IF NOT EXISTS quality_report (
  id INTEGER PRIMARY KEY,
  report_ts TEXT NOT NULL DEFAULT (datetime('now')),
  severity TEXT NOT NULL CHECK(severity IN ('INFO','WARN','ERROR')),
  message TEXT NOT NULL,
  context_json TEXT
);
```

DB erzeugen:

```bash
sqlite3 data/db.sqlite3 < data/schema.sql
```

---

## 4) Kategorien seeden

`data/seed_categories.sql`:

```sql
INSERT OR IGNORE INTO category(key,name,points_total,sort_index) VALUES
 ('GRUNDRECHENARTEN','Grundrechenarten',20,1),
 ('ZAHLENRAUM','Zahlenraum',20,2),
 ('TEXTAUFGABEN','Textaufgaben',20,3),
 ('BRUECHE_GLEICHUNGEN','Brüche und Gleichungen',20,4),
 ('RAUMVORSTELLUNG','Raumvorstellung',20,5);
```

Ausführen:

```bash
sqlite3 data/db.sqlite3 < data/seed_categories.sql
```

---

## 5) **Deine Regelspezifikation** als JSON ablegen & einspielen

1. **Datei anlegen:** `data/rules/global.json`
   → Struktur gespiegelt zu deiner Spezifikation:

```json
{
  "fundamental": {
    "total_points": 100,
    "categories": 5,
    "points_per_category": 20,
    "duration_min": 90,
    "pass_points": 60
  },
  "grundrechenarten": { "tasks": [ /* GRA-a .. GRA-f (jeweils expr/vars/rules/points) */ ] },
  "zahlenraum":       { "tasks": [ /* ZR-a .. ZR-f */ ] },
  "textaufgaben":     { "tasks": [ /* TX-a .. TX-b */ ] },
  "brueche_gleichungen": { "tasks": [ /* BG-a .. BG-e */ ] },
  "raumvorstellung":  { "tasks": [ /* RV-a .. RV-b */ ] },
  "qualitaet": {
    "similarity":   { "window":3, "max_identical":0.7, "max_regens":5 },
    "plausibility": { "salary_eur":[1000,5000], "work_hours_week":[20,50], "max_single_den":30, "result_abs_max":1000000 },
    "math":         { "round_dp":2, "reduce_fractions":true }
  }
}
```

> **Tipp:** Jede einzelne Regel aus deinem (langen) Regeltext findet hier Platz – falls etwas nicht natürlich als Key/Value passt, packe es als `notes`-String in die jeweilige Task.

2. **Importer:** `src/load_rules.py`

```python
# src/load_rules.py
import json, sqlite3
from pathlib import Path

DB = 'data/db.sqlite3'
RULES = Path('data/rules/global.json')

def main():
    con = sqlite3.connect(DB)
    cur = con.cursor()
    obj = json.loads(RULES.read_text(encoding='utf-8'))
    for section, payload in obj.items():
        if isinstance(payload, dict):
            for k, v in payload.items():
                cur.execute(
                    'INSERT INTO rule_global(section,key,value_json) VALUES (?,?,?)',
                    (section, k, json.dumps(v, ensure_ascii=False))
                )
        else:
            cur.execute(
                'INSERT INTO rule_global(section,key,value_json) VALUES (?,?,?)',
                (section, 'value', json.dumps(payload, ensure_ascii=False))
            )
    con.commit(); con.close()
    print('OK: rules imported')

if __name__ == '__main__':
    main()
```

Ausführen:

```bash
python src/load_rules.py
```

---

## 6) DOCX-Import (Rohtext + eingebettete Bilder → DB)

`src/import_docx.py`:

```python
import zipfile, io, hashlib
from pathlib import Path
import sqlite3
import docx  # python-docx
from PIL import Image
from tqdm import tqdm

DB = 'data/db.sqlite3'
ASSETS_DIR = Path('data/assets')

def sha256_bytes(data: bytes) -> str:
    h = hashlib.sha256(); h.update(data); return h.hexdigest()

def save_bytes(path: Path, data: bytes):
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_bytes(data)

def import_one_docx(docx_path: Path, con: sqlite3.Connection):
    cur = con.cursor()

    raw = docx_path.read_bytes()
    cur.execute(
        'INSERT INTO provenance(source_doc,sha256,bytes) VALUES (?,?,?)',
        (docx_path.name, sha256_bytes(raw), len(raw))
    )

    # 1) Textblöcke (Paragraphs + Tabellen)
    d = docx.Document(docx_path)
    block_index = 0

    for p in d.paragraphs:
        text = (p.text or '').strip()
        if text:
            cur.execute(
                'INSERT INTO source_block(source_doc,block_index,kind,text) VALUES (?,?,?,?)',
                (docx_path.name, block_index, 'paragraph', text)
            )
            block_index += 1

    for t in d.tables:
        # Tabelle als Zeilen mit Pipe-Trenner
        rows = []
        for r in t.rows:
            rows.append(' | '.join([c.text.strip() for c in r.cells]))
        table_text = '\n'.join(rows).strip()
        if table_text:
            cur.execute(
                'INSERT INTO source_block(source_doc,block_index,kind,text) VALUES (?,?,?,?)',
                (docx_path.name, block_index, 'table', table_text)
            )
            block_index += 1

    # 2) Eingebettete Bilder aus dem DOCX-ZIP (word/media/*)
    with zipfile.ZipFile(docx_path, 'r') as z:
        media_names = [n for n in z.namelist() if n.startswith('word/media/')]
        for m in media_names:
            data = z.read(m)
            digest = sha256_bytes(data)
            rel = ASSETS_DIR / docx_path.stem / Path(m).name  # data/assets/<docname>/<img.png>
            save_bytes(rel, data)

            width = height = None
            try:
                with Image.open(io.BytesIO(data)) as im:
                    width, height = im.size
            except Exception:
                pass

            cur.execute(
                'INSERT OR IGNORE INTO asset(kind,rel_path,sha256,width_px,height_px,alt) VALUES (?,?,?,?,?,?)',
                ('image', str(rel), digest, width, height, f'Image from {docx_path.name}')
            )

    con.commit()

def main(folder: str):
    con = sqlite3.connect(DB)
    con.execute('PRAGMA foreign_keys=ON')
    files = sorted(Path(folder).glob('*.docx'))
    for f in tqdm(files, desc='Import DOCX'):
        import_one_docx(f, con)
    con.close()
    print('OK: import complete')

if __name__ == '__main__':
    import argparse
    ap = argparse.ArgumentParser()
    ap.add_argument('folder', help='Ordner mit .docx')
    args = ap.parse_args()
    main(args.folder)
```

Ausführen (auf deinen Test-Ordner zeigen):

```bash
python src/import_docx.py ./tests_raw
```

---

## 7) Datenbank-Checks (nur Datenbasis, keine Generierung)

`src/validator.py`:

```python
import sqlite3
from tabulate import tabulate

DB = 'data/db.sqlite3'

def expect(cond, msg):
    if not cond:
        raise AssertionError(msg)

def check_categories(cur, rows):
    expect(len(rows) == 5, 'Es müssen 5 Kategorien existieren.')
    expect(all(r[2] == 20 for r in rows), 'Jede Kategorie muss 20 Punkte haben.')

def main():
    con = sqlite3.connect(DB)
    cur = con.cursor()

    checks = []

    rows = cur.execute('SELECT key,name,points_total FROM category ORDER BY sort_index').fetchall()
    try:
        check_categories(cur, rows)
        checks.append(('categories', 'PASS', f'{len(rows)} Kategorien, je 20 Punkte'))
    except AssertionError as e:
        checks.append(('categories', 'FAIL', str(e)))

    n_rules = cur.execute('SELECT COUNT(*) FROM rule_global').fetchone()[0]
    checks.append(('rules_present', 'PASS' if n_rules > 0 else 'FAIL', f'{n_rules} Regelzeilen'))

    n_assets = cur.execute('SELECT COUNT(*) FROM asset').fetchone()[0]
    checks.append(('assets_count', 'INFO', f'{n_assets} Assets registriert'))

    n_src = cur.execute('SELECT COUNT(*) FROM source_block').fetchone()[0]
    checks.append(('source_blocks', 'INFO', f'{n_src} Textblöcke'))

    print(tabulate(checks, headers=['Check','Status','Details']))
    con.close()

if __name__ == '__main__':
    main()
```

Ausführen:

```bash
python src/validator.py
```

---

## 8) Schnell-Statistiken (direkt per sqlite3)

```bash
sqlite3 data/db.sqlite3 "
SELECT 'categories', COUNT(*) FROM category UNION ALL
SELECT 'rule_global', COUNT(*) FROM rule_global UNION ALL
SELECT 'source_block', COUNT(*) FROM source_block UNION ALL
SELECT 'asset', COUNT(*) FROM asset;"
```

---

## 9) (Optional) **Template-Skeleton** seeden (ohne Generierung)

Wenn du dem Generator später **strukturiert** Vorlagen geben willst, lege YAML an – **nur DB-Seite**:

`data/templates.yaml` (Ausschnitt):

```yaml
- category: GRUNDRECHENARTEN
  items:
    - code: GRA-a
      title: Addition/Subtraktion
      expr: "{a} + {b} - {c} ="
      difficulty: 1
      result_min: -100
      result_max: 220
      vars:
        a: { domain_type: RANGE, domain: {min:50, max:150, int:true} }
        b: { domain_type: RANGE, domain: {min:20, max:80, int:true} }
        c: { domain_type: RANGE, domain: {min:10, max:50, int:true} }
      rules:
        - { rule_key: round_2dp, params: {} }
# ... weitere Items gemäß deiner Spezifikation ...
```

`src/seed_templates_from_yaml.py`:

```python
import yaml, json, sqlite3
from pathlib import Path

DB='data/db.sqlite3'

def upsert_template(cur, category_key, code, title, expr, diff, rmin, rmax, notes):
    cat_id = cur.execute('SELECT id FROM category WHERE key=?', (category_key,)).fetchone()[0]
    cur.execute('INSERT OR IGNORE INTO template(category_id,code,title,expr,difficulty,result_min,result_max,notes) VALUES (?,?,?,?,?,?,?,?)',
                (cat_id, code, title, expr, diff, rmin, rmax, notes))
    cur.execute('UPDATE template SET title=?, expr=?, difficulty=?, result_min=?, result_max=?, notes=? WHERE code=?',
                (title, expr, diff, rmin, rmax, notes, code))

def main():
    db = sqlite3.connect(DB)
    cur = db.cursor()
    groups = yaml.safe_load(Path('data/templates.yaml').read_text(encoding='utf-8'))
    for g in groups:
        cat = g['category']
        for it in g['items']:
            upsert_template(cur, cat, it['code'], it['title'], it['expr'], it['difficulty'], it.get('result_min'), it.get('result_max'), it.get('notes'))
            t_id = cur.execute('SELECT id FROM template WHERE code=?', (it['code'],)).fetchone()[0]
            cur.execute('DELETE FROM template_var WHERE template_id=?', (t_id,))
            cur.execute('DELETE FROM template_rule WHERE template_id=?', (t_id,))
            for vname, spec in (it.get('vars') or {}).items():
                # spec kann {domain_type, domain} sein oder direkt domain
                if 'domain_type' in spec:
                    domain_type = spec['domain_type']
                    domain = spec.get('domain', spec)
                else:
                    # Fallback: RANGE angenommen
                    domain_type = 'RANGE'
                    domain = spec
                cur.execute('INSERT INTO template_var(template_id,name,domain_type,domain_json) VALUES (?,?,?,?)',
                            (t_id, vname, domain_type, json.dumps(domain, ensure_ascii=False)))
            for rule in (it.get('rules') or []):
                cur.execute('INSERT INTO template_rule(template_id,rule_key,params_json) VALUES (?,?,?)',
                            (t_id, rule['rule_key'], json.dumps(rule.get('params',{}), ensure_ascii=False)))
    db.commit(); db.close(); print('OK: templates seeded')

if __name__ == '__main__':
    main()
```

Seeden:

```bash
python src/seed_templates_from_yaml.py
```

---

## 10) (Optional) Assets manuell Templates zuordnen

Falls in den DOCX Bilder zu bestimmten Vorlagen gehören:

* CSV `data/map_template_assets.csv` mit Spalten: `code,rel_path`
  (relativer Pfad wie in `asset.rel_path`, z. B. `data/assets/TestHeft1/image1.png`)

`src/link_assets.py`:

```python
import csv, sqlite3

con = sqlite3.connect('data/db.sqlite3')
cur = con.cursor()
with open('data/map_template_assets.csv', newline='', encoding='utf-8') as f:
    reader = csv.reader(f)
    for code, rel in reader:
        t = cur.execute('SELECT id FROM template WHERE code=?', (code,)).fetchone()
        a = cur.execute('SELECT id FROM asset WHERE rel_path=?', (rel,)).fetchone()
        if t and a:
            cur.execute('INSERT OR IGNORE INTO template_asset(template_id,asset_id) VALUES (?,?)', (t[0], a[0]))
con.commit(); con.close(); print('OK: assets linked')
```

Ausführen:

```bash
python src/link_assets.py
```

---

## 11) Integritätsreport & Snapshot

```bash
sqlite3 data/db.sqlite3 "
SELECT 'templates', COUNT(*) FROM template UNION ALL
SELECT 'vars', COUNT(*) FROM template_var UNION ALL
SELECT 'rules', COUNT(*) FROM template_rule UNION ALL
SELECT 'assets', COUNT(*) FROM asset UNION ALL
SELECT 'source_blocks', COUNT(*) FROM source_block UNION ALL
SELECT 'rule_global', COUNT(*) FROM rule_global;
"
```

(Optional) Git Versionierung:

```bash
git init
printf "*.sqlite3\n" > .gitignore
# Optional: git lfs install && git lfs track "data/assets/*"
git add .
git commit -m "DB-Schema, Regeln, DOCX-Import, Validierung"
cp data/db.sqlite3 data/db-$(date +%Y%m%d).sqlite3
```
