# n8n Workflow – Walona Group Selbstauskunft

Build-Spezifikation für einen n8n-Workflow, der beim Absenden der Selbstauskunft
die Daten empfängt, ein **PDF generiert (via PDFShift)**, es **per E-Mail** verschickt
und die Daten in **Monday.com** synchronisiert.

Diese Datei ist als Vorlage gedacht, die einer KI / einem Entwickler übergeben werden
kann, um den Workflow zu bauen.

---

## 1. Architektur (Überblick)

```
[Formular Selbstauskunft]
        │  HTTP POST (JSON)
        ▼
[1) Webhook Trigger]  ──▶  [2) Set / Normalisieren]
        │
        ├──▶ [3) HTML für PDF bauen (Code)] ──▶ [4) PDFShift → PDF] ──▶ [5) E-Mail mit PDF]
        │
        └──▶ [6) Monday.com Item anlegen]  (+ optional [7) CRM / OneDrive nach Herkunft])
```

Das Formular sendet den JSON-Payload per `POST` an die Webhook-URL. Die URL wird im
Formular über das Prop `webhookUrl` gesetzt (siehe Abschnitt 8).

---

## 2. Node 1 – Webhook Trigger

- **Node:** `Webhook`
- **Method:** `POST`
- **Path:** z. B. `selbstauskunft`
- **Response:** `Respond immediately` (Code 200) oder „Using Respond to Webhook Node"
- Der komplette Payload liegt danach unter `{{ $json.body }}` (bei „Response immediately")
  bzw. `{{ $json }}` – je nach n8n-Version. Im Folgenden wird `$json` als Wurzel des
  Payloads verwendet; ggf. `.body` voranstellen.

> **Wichtig:** Beim ersten Testlauf einmal echte Daten aus dem Formular schicken und die
> Struktur „pinnen", damit alle Felder im Mapping auswählbar sind.

---

## 3. Payload-Schema (Vertrag Formular → Workflow)

Der Payload ist **immer gleich aufgebaut**. Beträge sind bereits als **Zahlen** (Monatswerte
und Summen) berechnet – es muss nichts nachgerechnet werden. Alle Geldbeträge sind in der
Währung aus `waehrung` zu interpretieren.

```jsonc
{
  "meta": { "formVersion": "1.0", "submittedAt": "2026-07-07T10:00:00.000Z", "brand": "Walona Group" },

  "herkunft": "DE",          // "DE" | "AT" | "CH"  → für CRM-Routing
  "waehrung": "EUR",         // "EUR" | "CHF"       → CH ⇒ CHF, sonst EUR

  "kundenprotokoll": {
    "berater": "", "portfolioManager": "",
    "gespraechsnotizen": "", "naechsteSchritte": ""
  },

  "applicants": [
    {
      "anrede": "", "vorname": "", "nachname": "",
      "geburtsdatum": "", "geburtsort": "", "geburtsland": "",
      "telefon": "", "email": "",
      "adresse": { "strasse": "", "hausnummer": "", "plz": "", "ort": "", "wohnhaftSeit": "" },
      "familienstand": "", "guterstand": "",
      "nationalitaeten": ["Deutschland"],
      "aufenthaltstitel": "",
      "beschaeftigung": {
        "art": "", "nettoeinkommenMonat": 0, "beschaeftigtSeit": "",
        "gehaelterProJahr": 12, "arbeitgeber": "", "beruf": "",
        "nebenjob": "", "nebenjobEinkommenMonat": 0, "nebenjobBeschaeftigtSeit": ""
      }
    }
  ],

  "kinder": {
    "vorhanden": false, "anzahl": 0, "beschreibung": "",
    "liste": [
      { "geburtsdatum": "", "kindergeld": false, "kindergeldBetragMonat": 0,
        "unterhalt": false, "unterhaltBetragMonat": 0 }
    ]
  },

  "finanzen": {
    "vermoegenEinkommen": {
      "eigenkapital": 0, "bankSparguthaben": 0, "wertpapiere": 0,
      "lebensversicherungRueckkaufswert": 0, "lebensversicherungBeitragMonat": 0,
      "bausparAngespart": 0, "bausparBeitragMonat": 0,
      "sonstigeEinkuenfteMonat": 0, "gesamteinnahmenMonat": 0
    },
    "ausgaben": {
      "warmmieteMonat": 0,
      "unterhaltsverpflichtungen": false, "unterhaltsverpflichtungenMonat": 0,
      "privateKrankenversicherung": false, "privateKrankenversicherungMonat": 0,
      "sonstigeAusgabenMonat": 0, "gesamtausgabenMonat": 0
    },
    "privateKredite": {
      "antragsteller":    [ { "restschuld": 0, "darlehenshoeheGesamt": 0, "rateMonat": 0, "restlaufzeit": "" } ],
      "mitantragsteller": [ ]
    },
    "kommentar": ""
  },

  "immobilien": [
    {
      "anschrift": "", "objektarten": [], "gewerbeart": "",
      "gesamtwohnflaecheM2": 0, "vermieteteWohnflaecheM2": 0, "gewerblicheNutzflaecheM2": 0,
      "baujahr": "", "kaufpreis": 0, "wertHeute": 0, "kaltmieteMonat": 0, "anzahlWohneinheiten": 0,
      "verbindlichkeiten": [
        { "darlehen": 1, "kreditgeber": "", "lebensversicherungsgesellschaft": "",
          "eingetrageneGrundschulden": 0, "aktuellerDarlehensstand": 0,
          "sollzinsProzent": 0, "tilgungProzent": 0, "zinsbindungBis": "", "monatlicheRate": 0 }
      ],
      "summeDarlehensstand": 0, "summeMonatlicheBelastung": 0,
      "beschreibung": "", "besonderheiten": ""
    }
  ]
}
```

`applicants` enthält 1–2 Einträge, `immobilien` 0–n Einträge.

---

## 4. Node 2 – Set / Normalisieren (optional, empfohlen)

Kleine Hilfsfelder für die weiteren Nodes bereitstellen:

| Feld | Ausdruck |
|------|----------|
| `empfaengerEmail` | `{{ $json.applicants[0].email }}` |
| `kundenName`      | `{{ $json.applicants[0].vorname }} {{ $json.applicants[0].nachname }}` |
| `waehrung`        | `{{ $json.waehrung }}` |
| `dateiName`       | `Selbstauskunft_{{ $json.applicants[0].nachname || 'Kunde' }}.pdf` |

---

## 5. Node 3 – HTML für das PDF bauen (Code-Node)

PDFShift benötigt **HTML** als Input. Es gibt zwei Wege:

### Variante A (empfohlen für 1:1-Design)
Das Formular kann das **fertig gerenderte PDF-HTML** direkt als Feld `pdfHtml` mitsenden
(dieselbe Vorlage wie beim Button „Als PDF herunterladen", inkl. Walona-Design & TÜV-Siegel).
Dann entfällt Node 3 komplett und PDFShift bekommt `source = {{ $json.pdfHtml }}`.
> Diese Payload-Erweiterung ist eine kleine Anpassung im Formular – bei Bedarf ergänzen lassen.

### Variante B (HTML im Workflow bauen)
Solange `pdfHtml` nicht mitgesendet wird, baut ein **Code-Node** das HTML aus dem JSON.
Beispiel (JavaScript, gibt `html` als Feld zurück):

```javascript
const d = $json.body || $json;
const cur = d.waehrung === 'CHF' ? 'CHF' : '€';
const eur = (n) => (Number(n)||0).toLocaleString(d.waehrung === 'CHF' ? 'de-CH' : 'de-DE') + ' ' + cur;
const esc = (s) => String(s==null?'':s).replace(/[&<>]/g, m => ({'&':'&amp;','<':'&lt;','>':'&gt;'}[m]));
const row = (k,v) => v ? `<tr><td style="color:#8a8168;padding:3px 12px 3px 0;">${esc(k)}</td><td style="font-weight:600;">${esc(v)}</td></tr>` : '';
const sec = (t) => `<h2 style="font:600 14px Arial;text-transform:uppercase;letter-spacing:.06em;color:#161310;border-bottom:2px solid #C8A701;padding-bottom:4px;margin:22px 0 8px;">${t}</h2>`;

const a = d.applicants[0] || {};
let h = `<div style="font-family:Arial,sans-serif;color:#1a1712;width:760px;padding:8px;">
  <div style="display:flex;align-items:center;gap:12px;border-bottom:3px solid #C8A701;padding-bottom:12px;margin-bottom:6px;">
    <div style="width:38px;height:38px;border-radius:9px;background:#161310;color:#C8A701;display:flex;align-items:center;justify-content:center;font:700 20px Arial;">W</div>
    <div><div style="font:700 17px Arial;letter-spacing:.12em;">WALONA GROUP</div>
    <div style="font:400 11px Arial;color:#8a8168;">Selbstauskunft · ${new Date().toLocaleDateString('de-DE')} · Herkunft: ${esc(d.herkunft)} · Währung: ${esc(d.waehrung)}</div></div>
  </div>`;

// Antragsteller
d.applicants.forEach((a,i)=>{
  h += sec('Antragsteller '+(i+1)) + '<table style="font:400 12px Arial;border-collapse:collapse;">'
    + row('Name',[a.anrede,a.vorname,a.nachname].filter(Boolean).join(' '))
    + row('Geburtsdatum',a.geburtsdatum)
    + row('E-Mail',a.email) + row('Telefon',a.telefon)
    + row('Meldeadresse',[a.adresse.strasse,a.adresse.hausnummer].filter(Boolean).join(' ')+', '+[a.adresse.plz,a.adresse.ort].filter(Boolean).join(' '))
    + row('Familienstand',a.familienstand+(a.guterstand?' ('+a.guterstand+')':''))
    + row('Staatsangehörigkeit',(a.nationalitaeten||[]).join(', '))
    + row('Beschäftigung',a.beschaeftigung.art)
    + row('Nettoeinkommen/Monat', a.beschaeftigung.nettoeinkommenMonat ? eur(a.beschaeftigung.nettoeinkommenMonat) : '')
    + row('Arbeitgeber',a.beschaeftigung.arbeitgeber) + row('Beruf',a.beschaeftigung.beruf)
    + '</table>';
});

// Finanzen
const f = d.finanzen;
h += sec('Finanzielle Situation') + '<table style="font:400 12px Arial;border-collapse:collapse;">'
  + row('Gesamteinnahmen/Monat', eur(f.vermoegenEinkommen.gesamteinnahmenMonat))
  + row('Warmmiete/Monat', f.ausgaben.warmmieteMonat ? eur(f.ausgaben.warmmieteMonat) : '')
  + row('Gesamtausgaben/Monat', eur(f.ausgaben.gesamtausgabenMonat))
  + row('Überschuss/Monat', eur((f.vermoegenEinkommen.gesamteinnahmenMonat||0) - (f.ausgaben.gesamtausgabenMonat||0)))
  + '</table>';

// Immobilien
d.immobilien.forEach((im,i)=>{
  h += sec('Immobilie '+(i+1)) + '<table style="font:400 12px Arial;border-collapse:collapse;">'
    + row('Anschrift',im.anschrift) + row('Objektart',(im.objektarten||[]).join(', '))
    + row('Kaufpreis', im.kaufpreis ? eur(im.kaufpreis) : '')
    + row('Wert heute', im.wertHeute ? eur(im.wertHeute) : '')
    + row('Darlehensstand ges.', eur(im.summeDarlehensstand))
    + row('Monatliche Belastung', eur(im.summeMonatlicheBelastung))
    + '</table>';
});

h += `<div style="margin-top:24px;font:400 9px Arial;color:#a49a80;border-top:1px solid #eee;padding-top:8px;">Walona Group · Vertraulich · Erstellt am ${new Date().toLocaleString('de-DE')}</div></div>`;

return [{ json: { html: h } }];
```

---

## 6. Node 4 – PDFShift (HTTP Request)

- **Node:** `HTTP Request`
- **Method:** `POST`
- **URL:** `https://api.pdfshift.io/v3/convert/pdf`
- **Authentication:** Basic Auth → **User:** `api`, **Password:** `DEIN_PDFSHIFT_API_KEY`
  (oder Header `X-API-Key: DEIN_KEY`)
- **Response Format:** `File` / `Binary` (Property z. B. `data`)
- **Body (JSON):**

```json
{
  "source": "={{ $json.html }}",
  "landscape": false,
  "format": "A4",
  "margin": "12mm",
  "use_print": true
}
```

> Bei Variante A: `"source": "={{ $json.pdfHtml }}"`.

Ergebnis ist die PDF-Datei als Binärdaten (für den E-Mail-Anhang).

---

## 7. Node 5 – E-Mail mit PDF-Anhang

- **Node:** `Send Email` (SMTP) oder `Gmail`
- **To:** `{{ $json.empfaengerEmail }}` (Kunde) — optional zusätzlich feste Walona-Adresse als BCC
- **Subject:** `Ihre Selbstauskunft – Walona Group`
- **Text/HTML:** kurzer Begleittext
- **Attachments:** die Binärdaten aus dem PDFShift-Node (Property `data`),
  Dateiname `{{ $json.dateiName }}`

> Reihenfolge beachten: Der E-Mail-Node muss die Binärdaten des PDFShift-Nodes „sehen".
> Ggf. mit einem `Merge`-Node die Binärdaten + Felder zusammenführen.

---

## 8. Node 6 – Monday.com Item anlegen

- **Node:** `Monday.com` → *Create Item* (oder `GraphQL`/HTTP mit Monday API)
- **Board:** Ziel-Board wählen
- **Item Name:** `{{ $json.kundenName }}`
- **Spalten-Mapping (Beispiel):**

| Monday-Spalte            | Wert (Ausdruck) |
|--------------------------|-----------------|
| Herkunft (DE/AT/CH)      | `{{ $json.herkunft }}` |
| Währung                  | `{{ $json.waehrung }}` |
| Berater                  | `{{ $json.kundenprotokoll.berater }}` |
| Portfolio Manager        | `{{ $json.kundenprotokoll.portfolioManager }}` |
| E-Mail                   | `{{ $json.applicants[0].email }}` |
| Telefon                  | `{{ $json.applicants[0].telefon }}` |
| Nettoeinkommen/Monat     | `{{ $json.applicants[0].beschaeftigung.nettoeinkommenMonat }}` |
| Gesamteinnahmen/Monat    | `{{ $json.finanzen.vermoegenEinkommen.gesamteinnahmenMonat }}` |
| Gesamtausgaben/Monat     | `{{ $json.finanzen.ausgaben.gesamtausgabenMonat }}` |
| Anzahl Immobilien        | `{{ $json.immobilien.length }}` |
| Anzahl Kinder            | `{{ $json.kinder.anzahl }}` |
| Eingegangen am           | `{{ $json.meta.submittedAt }}` |

---

## 9. Node 7 – CRM / OneDrive (optional, nach Herkunft)

Für die bestehende CRM-Automatisierung (OneDrive-Ordner bei Kundenname) kann anhand
`herkunft` (DE/AT/CH) unterschiedlich geroutet werden – z. B. mit einem `Switch`-Node:

- `herkunft === "DE"` → Prozess Deutschland
- `herkunft === "AT"` → Prozess Österreich
- `herkunft === "CH"` → Prozess Schweiz (CHF)

---

## 10. Benötigte Credentials / Kosten

| Dienst      | Zweck               | Hinweis |
|-------------|---------------------|---------|
| PDFShift    | HTML → PDF          | API-Key erforderlich (Free-Tier für Tests vorhanden) |
| SMTP/Gmail  | Versand der PDF     | z. B. Firmen-Postfach |
| Monday.com  | Datensynchronisation| API-Token (persönlich oder Service-Account) |
| n8n Cloud/Self-Host | Ausführung  | Self-Host ~28 €/Monat, alternativ n8n Cloud |

---

## 11. Formular-Konfiguration (Webhook-URL setzen)

Im Formular `Selbstauskunft.dc.html` wird die Webhook-URL über das Prop `webhookUrl`
gesetzt (im `data-props` des `<script type="text/x-dc" …>`). Ist keine URL gesetzt, läuft
das Formular im **Demo-Modus** (kein Versand, aber vollständig strukturierte Daten).

Beispiel `data-props`:

```json
{ "webhookUrl": "https://DEIN-N8N/webhook/selbstauskunft" }
```

---

## 12. Test-Payload (zum Import / Pinnen in n8n)

```json
{
  "meta": { "formVersion": "1.0", "submittedAt": "2026-07-07T10:00:00.000Z", "brand": "Walona Group" },
  "herkunft": "CH", "waehrung": "CHF",
  "kundenprotokoll": { "berater": "Max Berater", "portfolioManager": "Lena Closer", "gespraechsnotizen": "Erstkontakt", "naechsteSchritte": "Unterlagen anfordern" },
  "applicants": [
    { "anrede": "Herr", "vorname": "Max", "nachname": "Mustermann", "geburtsdatum": "1985-04-12",
      "telefon": "+41 79 000 00 00", "email": "max@beispiel.ch",
      "adresse": { "strasse": "Bahnhofstrasse", "hausnummer": "1", "plz": "8001", "ort": "Zürich", "wohnhaftSeit": "2015-01-01" },
      "familienstand": "Verheiratet", "guterstand": "Ohne Gütertrennung", "nationalitaeten": ["Schweiz"], "aufenthaltstitel": "",
      "beschaeftigung": { "art": "Angestellt", "nettoeinkommenMonat": 8500, "beschaeftigtSeit": "2016-03-01", "gehaelterProJahr": 13, "arbeitgeber": "Muster AG", "beruf": "Ingenieur", "nebenjob": "", "nebenjobEinkommenMonat": 0, "nebenjobBeschaeftigtSeit": "" } }
  ],
  "kinder": { "vorhanden": true, "anzahl": 1, "beschreibung": "", "liste": [ { "geburtsdatum": "2018-06-01", "kindergeld": true, "kindergeldBetragMonat": 250, "unterhalt": false, "unterhaltBetragMonat": 0 } ] },
  "finanzen": {
    "vermoegenEinkommen": { "eigenkapital": 120000, "bankSparguthaben": 40000, "wertpapiere": 30000, "lebensversicherungRueckkaufswert": 0, "lebensversicherungBeitragMonat": 0, "bausparAngespart": 0, "bausparBeitragMonat": 0, "sonstigeEinkuenfteMonat": 0, "gesamteinnahmenMonat": 8750 },
    "ausgaben": { "warmmieteMonat": 2200, "unterhaltsverpflichtungen": false, "unterhaltsverpflichtungenMonat": 0, "privateKrankenversicherung": true, "privateKrankenversicherungMonat": 450, "sonstigeAusgabenMonat": 600, "gesamtausgabenMonat": 3250 },
    "privateKredite": { "antragsteller": [ { "restschuld": 5000, "darlehenshoeheGesamt": 15000, "rateMonat": 300, "restlaufzeit": "18 Monate" } ], "mitantragsteller": [] },
    "kommentar": ""
  },
  "immobilien": [
    { "anschrift": "Seestrasse 10, 8002 Zürich", "objektarten": ["Mehrfamilienhaus"], "gewerbeart": "",
      "gesamtwohnflaecheM2": 220, "vermieteteWohnflaecheM2": 160, "gewerblicheNutzflaecheM2": 0, "baujahr": "1998",
      "kaufpreis": 1500000, "wertHeute": 1800000, "kaltmieteMonat": 4800, "anzahlWohneinheiten": 3,
      "verbindlichkeiten": [ { "darlehen": 1, "kreditgeber": "Kantonalbank", "lebensversicherungsgesellschaft": "", "eingetrageneGrundschulden": 900000, "aktuellerDarlehensstand": 850000, "sollzinsProzent": 1.4, "tilgungProzent": 2, "zinsbindungBis": "2030-01-01", "monatlicheRate": 2400 } ],
      "summeDarlehensstand": 850000, "summeMonatlicheBelastung": 2400, "beschreibung": "", "besonderheiten": "" }
  ]
}
```
