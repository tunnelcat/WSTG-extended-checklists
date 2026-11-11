# WSTG Pentester Quick Reference

A condensed companion to the **[OWASP Web Security Testing Guide (WSTG) v4.2+](https://owasp.org/www-project-web-security-testing-guide/)**. It gives you what to test and how to test it per WSTG entry, so you don't have to re-read the full guide during an engagement.

This is a derivative work based on the OWASP WSTG and is released under the same license as the original, Creative Commons Attribution-ShareAlike 4.0. See [Attribution & License](#attribution--license) below.

## Contents

| File | What it is |
|------|------------|
| `WSTG_QuickRef.md` | Per-entry quick reference (WSTG-INFO-01 through WSTG-WASM-01). Each finding has a status checklist (Not Started / Pass / Issues / N/A), a "Look For" summary, copy-paste commands and payloads in syntax-highlighted code blocks, relevant tools, and a Notes section to fill in. |
| `WSTG_Checklist_Extended.xlsx` | The OWASP WSTG testing checklist spreadsheet with three added columns, "Look For", "Reference" (quick commands), and "Tools", inserted between the Status and Notes columns. The original categories, objectives, hyperlinks, and formatting are preserved. Use it as a live engagement tracker. |

Both files cover every WSTG category: Information Gathering, Configuration & Deployment, Identity Management, Authentication, Authorization, Session Management, Injection, Error Handling, Weak Cryptography, Business Logic, Client-side, API Testing, and WebAssembly.

## Usage

Open `WSTG_QuickRef.md` in any Markdown viewer. On GitHub the checkboxes render as clickable task items.

Open `WSTG_Checklist_Extended.xlsx` in Excel, LibreOffice, or Google Sheets and tick off entries as you work through a target.

## Authorized testing only

These materials are for authorized security testing and education. Only test systems you own or have explicit written permission to assess. Unauthorized testing is illegal. The author assumes no liability for misuse.

## Attribution & License

This project adapts content from the **OWASP Web Security Testing Guide**, copyright the OWASP Foundation and WSTG contributors, licensed under **[Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/)**.

Original work: OWASP Web Security Testing Guide, https://owasp.org/www-project-web-security-testing-guide/

What this repository changed from the original: condensed "Look For" summaries per entry, added quick commands and tool references, added status checkboxes, and added three columns to the checklist spreadsheet (Look For, Reference, Tools).

Under the ShareAlike terms, this repository and all of its additions are also licensed under CC BY-SA 4.0. The full text is in the [`LICENSE`](LICENSE) file. If you reuse or adapt this material, give credit to both OWASP and this repository, indicate what you changed, and distribute your version under the same license.
