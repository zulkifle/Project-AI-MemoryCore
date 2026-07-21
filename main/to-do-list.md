# 📝 To-Do List
*Persistent list — study material, tasks, and reminders. Checked at every "Jessy" load until items are marked done.*

## Pending

- [ ] **API Gateway — System Design** — https://www.geeksforgeeks.org/system-design/what-is-api-gateway-system-design/
  - Added: 2026-07-16
- [ ] **Jenkins (official site)** — https://www.jenkins.io/
  - Added: 2026-07-16
- [ ] **Security Prompt (hamizi.net)** — https://security-prompt.hamizi.net/
  - Added: 2026-07-16
- [ ] **ST3 ACE Token SDK (MSC Trustgate)** — `C:\PROJECTS\MYTRUSTID DESKTOP\Token\ST3 ACE\msctrustgate-sdk.st3ace.standard.windows`
  - Added: 2026-07-16
- [ ] **MyTrustID Desktop — confirm minidriver-version causality** — roll failing laptop's Smart Card Mini Driver back from `2.0.17.503` to `2.0.17.107` (or update working laptop up to `503`) to prove/disprove it triggers the `WinSCard.dll` SEH-corruption crash; report finding to Longmai (token vendor) if confirmed
  - Added: 2026-07-22
  - Branch: `fix/rsa-keygen-crash-handling`
  - See `projects/active/mytrustid-desktop.md` session 2026-07-22 for full dump analysis

## Done
*(move items here with completion date once studied/completed)*

- [x] **MyTrustID Desktop — reproduce RSA-keygen crash locally** — got matching-hardware token, captured `.dmp` on failing laptop, analyzed via WinDbg — confirmed `BITNESS_MISMATCH_X86_INVALID_EXCEPTION_HANDLER` in `mytrustid_pkcs11.dll` → `WinSCard.dll` call chain
  - Completed: 2026-07-22

---
*Say "add to to-do list [item]" to add more, or "mark [item] done" to check one off.*
