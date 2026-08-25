# Petronas Java Migration — Open Questions for Petronas / PDB Digital

*Running log of questions that need answers from the client side (Izzuddin / PDB Digital, or whoever owns the Cardsys/CTS side) before certain design decisions can be finalized. Answered items move to the Answered section with the date and what was learned. See `petronas-java-migration.md` for full project detail — this file is just the question tracker.*

## Pending

- [ ] **CR-delimiter intentional or artifact?** — the new-format sample (`CardSys.A.260714.zip`, forwarded 2026-08-25) has every 394-byte card record followed by a `\r` (carriage return) byte; the old format has no delimiter at all (records concatenated back-to-back). Is this CR-delimiting a genuine new convention from Cardsys/CTS going forward, or an artifact of how this particular sample was exported/emailed to us? (If it's an artifact, we shouldn't build permanent handling around it.)
  - Added: 2026-08-25
- [ ] **What does payment mode `CC` represent?** — found in a new tag `NRCC` in the same sample. The old `BRS_Embossing File Format.pdf` only documents `PO` (Postpaid) and `PR` (Prepaid) for the Payment Mode position. What does `CC` mean functionally, and does it change any downstream signing/validation logic beyond just being a new recognized tag value?
  - Added: 2026-08-25
- [ ] **File Specification** (Pre-BRS item #1) — latest embossing file spec with all new values clearly identified. Old `BRS_Embossing File Format.pdf`/`Cardsys Data.docx` found on disk (2026-08-25) may serve as a starting point but predate the new values.
  - Added: 2026-08-24
- [ ] **Business Rules** (Pre-BRS item #3) — mapping/logic for each new value introduced in the embossing file.
  - Added: 2026-08-24
- [ ] **Documentation** (Pre-BRS item #4) — supporting technical documentation or process flow for the new values, beyond what old docs already cover.
  - Added: 2026-08-24
- [ ] **Sample Files — confirm sufficiency** (Pre-BRS item #2) — `CardSys.A.260714.zip` is a real sample with at least one new tag value (`NRCC`); confirm with Izzuddin whether this is *the* representative UAT sample, or if more/different new-value samples exist we should also check against.
  - Added: 2026-08-25
- [ ] **Rejected-record error-code taxonomy** — the legacy `.err` report has two error-code fields (`errorCode1`/`errorCode2`, e.g. real example `"02"`/`"401"` for a card-validity failure). We only know one real example; the full code list/meaning for other failure types (invalid length, unrecognized tag, signing failure, etc.) is unknown. `CardFileProcessor` currently emits a fixed placeholder (`"99"`/`"000"`) for every rejection regardless of cause — the human-readable description is still accurate, just not the numeric code. Needs the real taxonomy (from Cardsys/CTS or internal legacy documentation) before the Java `.err` output can be code-accurate.
  - Added: 2026-08-25
## Answered

- [x] **`.ssd` SeqNo — independent counter or the record's own embedded sequence field?**
  - Added: 2026-08-25, Answered: 2026-08-25 (self-resolved, no client input needed)
  - **Answer: independent counter, confirmed.** Compared records #2 and #3 of the real `NRPOC20260822_1.A.260824` sample: each record's own embedded `SEQUENCE_NUMBER` field (offset 24-30) reads `"200000"` and `"220000"` respectively, while the corresponding `.ssd` report lines' SeqNo field reads `"000002"`/`"000003"` — completely different values, ruling out any relationship between the two. The `.ssd` SeqNo is a simple 1-based counter assigned during report generation (matching the legacy `ulRecordNo = ulDataRecord++` pattern), unrelated to whatever the record's own embedded field represents. `CardFileProcessor`'s existing implementation (a simple 1-based-per-file counter) is confirmed correct as-is — no code change needed.
