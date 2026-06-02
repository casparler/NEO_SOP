# NEO-SOP – Projekt-Gedächtnis

> Single-Page-App zur antibiotischen Primärtherapie bei Verdacht auf
> Early-Onset-Sepsis (EOS) bei Früh- und Neugeborenen.
> Lokales Repo: `SOP Apps/` · Live: <https://casparler.github.io/NEO_SOP/>
> Quell-Repo: <https://github.com/casparler/NEO_SOP>

---

## 1. Aktueller Stand (Juni 2026)

| Datei | Zweck |
| --- | --- |
| `index.html` | UI, Eingabemaske, Output-Rendering, eingebettete `SOP_LOGIC` |
| `logic.js`   | Stand-alone Kopie der Logik-Engine (für Reuse/Tests) |
| `Flowchart-EOS-Therapie-mit-Medikamenten_Version_MAI2021.pdf` | Quell-SOP der Klinik (Stand 05/2021) |
| `.clinedocs/projectBrief.md`, `.clinedocs/kanban.md` | Ursprüngliches Briefing & Backlog |

**App-Features**
- Eingabe: Gewicht (g), SSW + Tage bei Geburt, Lebensalter (PNA, d).
- Situations-Chips: EOS-Primärtherapie · KNS/Katheter-Verdacht · V. a. Meningitis ·
  Blutkultur steril · Gezielte Therapie · V. a. Ureaplasmen · NVK/ESK einliegend.
- Berechnet aktuelles **PMA = GA-bei-Geburt + PNA** und leitet daraus
  Dosis & Intervall ab für: Ampicillin, Cefotaxim, Tobramycin, Vancomycin,
  Caspofungin, Fluconazol-Prophylaxe.
- Infobox: Haut-Desinfektion (Kodan / Octenidin / Betaisadona je nach SSW),
  Fluconazol-Status, Amphomoronal po.
- Tooltip mit Warnhinweisen (Hepatotoxizität, QT, CYP-Interaktionen,
  off-label-Status) bei Fluconazol-Prophylaxe.

---

## 2. Quellen & Hinterlegte Dosierungen

| Medikament | Quelle | Schema |
| --- | --- | --- |
| Ampicillin / Cefotaxim | SOP-Flowchart 05/2021 + Neofax 2009 | 50 mg/kg ED (Meningitis: 200 mg/kg/d ÷ 3 = 66,6 mg/kg q8h) · Intervall nach PMA × PNA |
| Tobramycin | Neofax 2009 | 4–5 mg/kg ED · Intervall 24–48 h nach PMA × PNA · Talspiegel < 2 mg/l vor 3. Gabe |
| Vancomycin | Klinik-SOP | 10 mg/kg ED (Meningitis 15 mg/kg) · Talspiegel-Ziel 5–10 (Meningitis 15–20) mg/l |
| Caspofungin | Klinik-SOP | bei < 28+0 SSW: Loading 25 mg/kg, dann 25 mg/kg q24h |
| Fluconazol-Prophylaxe | User-verifiziert 06.05.2026, off-label | 6 mg/kg · q72h (VLBW < 37 SSW & < 1 kg, PNA 0–14 d) bzw. q48h (PNA 14–28 d) |
| Clarithromycin (Ureaplasmen) | Oberarzt-Indikation | nur Hinweis, keine Dosis hinterlegt |

---

## 3. **WICHTIG – PMA-Klarstellung (Mail Sarina, Mai 2026)**

Hintergrund: Nach wiederholten Diskussionen zur Interpretation des Flowcharts.

> **Die Quelle für die Dosen ist das Neofax 2009.**
>
> "The antibiotic dosing charts reflect the fact that the renal function and drug
> elimination are most strongly correlated with Postmenstrual Age (**PMA**,
> equivalent to Gestational Age **plus** Neonatal Age). Postmenstrual age is
> therefore used as the **primary** determinant of dosing interval, with
> Postnatal Age as the **secondary** qualifier."
>
> **Beispiel:** Baby 28+0 SSW, jetzt 21 Tage alt → PMA 31 SSW (Zeile 30–36 SSW) →
> PNA 21 d (> 14) → Cefotaxim-Intervall **8 h**.
>
> **Heißt: Das aktuelle postmenstruelle Alter gilt – nicht das bei Geburt.**

### Konsequenzen für die App
- `SOP_LOGIC.calculatePMA(ssw_w, ssw_d, pna_d)` summiert bereits korrekt
  `GA + PNA` → die Rechenlogik ist **konform**.
- UI muss eindeutig sein, damit Nutzer nicht das Geburts-GA mit dem aktuellen
  PMA verwechseln:
  - Labels: **"SSW bei Geburt"**, **"Tage bei Geburt"**, **"Lebensalter (PNA, Tage)"**.
  - PMA-Badge zeigt **"Aktuelles PMA"** + Info-Tooltip mit Definition + Neofax-Zitat.
  - Footer/Header-Hinweis: "Dosierungs-Intervalle nach aktuellem PMA + PNA (Quelle: Neofax 2009)".
- Bei künftigen Tabellen-Reviews: Grenzen (28+6, 35+6, 43+6 SSW) immer als
  **PMA-Grenzen** dokumentieren, nie als Geburts-GA.

---

## 4. Architektur-Notizen

```diagram
╭─────────────────────╮      ╭────────────────────────╮
│ Inputs              │      │ SOP_LOGIC              │
│ • Gewicht           │─────▶│ • calculatePMA         │
│ • SSW bei Geburt    │      │ • getAmpiCefo          │
│ • PNA               │      │ • getTobramycin        │
│ • Situations-Chips  │      │ • getVancomycin        │
╰─────────────────────╯      │ • getFluconazol        │
                             │ • getInfobox           │
                             ╰───────────┬────────────╯
                                         │
                                         ▼
                             ╭────────────────────────╮
                             │ ui_update() in HTML    │
                             │ → Med-Cards + Alerts   │
                             │ → Infobox Prävention   │
                             ╰────────────────────────╯
```

- State liegt in `let state = { eos, vanco, meningitis, steril, targeted, ureaplasma, nvk_esk }`.
- `swapToVanco = state.targeted || state.vanco` schaltet Tobramycin → Vancomycin.
- `state.steril` blendet Cefotaxim & Caspofungin aus.
- `state.meningitis` erhöht Ampi/Cefo-Dosis und Vanco-Talspiegel-Ziel.

---

## 5. Backlog / Offene Punkte

- [ ] Fluconazol-Schema für **PNA > 28 d** in Quelle nachschlagen
      (aktuell Hinweis "Rücksprache OA").
- [ ] PWA-Features (Manifest, Service Worker, Offline-Cache) – aus Kanban noch offen.
- [ ] Unit-Tests für `SOP_LOGIC` (Edge-Cases: 28+6 ↔ 29+0, 35+6 ↔ 36+0, 43+6 ↔ 44+0).
- [ ] Doppelte Logik (in `index.html` **und** `logic.js`) konsolidieren – aktuell
      Gefahr von Drift. Vorschlag: nur noch `logic.js` als Quelle, in HTML per
      `<script src="logic.js">` einbinden.
- [ ] Konsistenz-Check Vancomycin-Intervalle vs. aktuelle Hauspolitik.
- [ ] Hinweis "Stand 05/2021 | Köln" prüfen, ggf. aktualisieren.

---

## 6. Konventionen für künftige Edits

- **Keine** stillen Dosis-Änderungen ohne Quelle: jede Anpassung mit Datum +
  Quellen-Kommentar im Code **und** Eintrag in diesem Dokument.
- PMA-Grenzwerte als Dezimalwerte (`28.85` = 28+6) lassen – Konvention bereits
  etabliert.
- UI-Texte deutsch, Code-Kommentare deutsch oder englisch (gemischt OK).
- Disclaimer "NUR FÜR ÄRZTLICHES PERSONAL" muss sichtbar bleiben.
