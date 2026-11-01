---
name: save-labs
description: Move and rename downloaded lab/medical test result PDFs into the health results archive, following the existing naming convention. Use when asked to file, save, move, or sort lab results / badania / wyniki from Downloads (or a zip) into the Wyniki folder.
version: 1.0.0
allowed-tools: Bash, Read
---

# Save lab results

Moves lab result PDFs (ALAB, Centrum Diagnostyki Laboratoryjnej, 4Active,
DiaBakter, etc.) from a source location (usually `~/Downloads`, often a
`*_results.zip` or an extracted folder) into the personal health archive at
`/home/uzer/syncpc/zdrowie/Wyniki`, renamed to match the archive's existing
naming convention.

## Naming pattern

```
YY-MM-DD-Lab-TestName.pdf
YY-MM-DD-TestName.pdf        (no lab prefix — only for the few labs that have
                               historically been filed without one, see below)
```

- **Date = "data i godz. pobrania" (sample collection date), not the order
  registration date or the report/execution date.** These often differ by
  several days for outsourced tests.
- Date is written `YY-MM-DD`, reversed from the PDF's `DD-MM-YYYY` format.
  E.g. pobranie `26-10-2026` → filename prefix `26-10-26`.
- If two files share the exact same date+lab+tests (e.g. two Morfologia
  reports same day), suffix with `1`/`2` (see `25-12-16-Alab-Morfologia1.pdf`
  / `...Morfologia2.pdf`).

## Workflow

1. **Find the source files.** If given a zip, unzip it first. List the PDFs.
2. **Extract text from each PDF** with `pdftotext file.pdf -` to read:
   - the lab name/header (determines the `Lab` prefix),
   - the "Materiał: ..., data i godz. pobrania: DD-MM-YYYY" line(s) — use
     the *earliest* pobranie date in the document for the filename,
   - the test/panel names actually reported (ignore lines like "Wyniki w
     późniejszym terminie" — those tests are NOT in this document, don't
     name the file after them).
   - If `pdftotext` fails or returns empty, the PDF may be encrypted. Try
     `qpdf --decrypt --password=<PESEL> in.pdf out.pdf` (lab portals often
     encrypt with the patient's PESEL) before giving up.
3. **Check existing naming conventions before inventing new abbreviations.**
   Run `ls /home/uzer/syncpc/zdrowie/Wyniki` and grep for similar past tests
   — reuse the exact short name already used (e.g. `Morfologia`,
   `Kalprotektyna`, `Lipidogram`, `ZelazoLipidogram`, `D` for witamina D,
   `Ferrytyna`, `CRP`, `Zelazo`, `KwasMoczowy`, `CholCalk`, `Glukoza`,
   `Pasozyty`, `OrganixGastro`, `Sibo`). Multiple tests in one PDF are
   joined with commas, no spaces: `CRP,Zelazo,CholCalk,D,Ferrytyna`. A
   combined iron+full-lipid-panel result is named `ZelazoLipidogram`, not
   spelled out.
4. **Lab prefix.** Known prefixes seen in the archive: `Alab` (ALAB
   Laboratoria), `4Active`, `DiaBakter`, `Borlamed`, `JakMotyl`, `ZJ`, `UM`.
   A few tests are historically filed **without** a lab prefix — notably
   Giardia lamblia / "Lamblie" results from Centrum Diagnostyki
   Laboratoryjnej (Dąbrowa Górnicza), and some one-off imaging reports
   (`GIMap`, `TKZatok`, `KandydozaKal`). When in doubt, check if a same-test
   file already exists in the archive and match its prefix style exactly.
5. **Move (don't copy)** each PDF directly to
   `/home/uzer/syncpc/zdrowie/Wyniki/<new-name>.pdf` with `mv`.
6. **Confirm with the user** before deleting the now-empty source folder or
   the original zip — only delete when explicitly asked.

## Example

A single ALAB order (`Ident. dokumentu zlecenia: AL26889073`) can split into
several separate PDFs by department (Hematologia, Biochemia, Immunochemia).
Treat each department PDF as its own file — don't try to merge them — but
reuse the same date prefix since they share one pobranie date:

```
26-10-26-Alab-Morfologia.pdf
26-10-26-Alab-CRP,ZelazoLipidogram,D,Ferrytyna.pdf
```

A third, separately-delivered PDF from a different lab for a test ordered
in the same batch (e.g. Giardia lamblia serology, reported later) keeps its
own historical no-prefix convention:

```
26-10-26-Lamblie.pdf
```
