# CyberChef Payments

**Payment cryptography workflows for engineers — inspectable, composable, shareable, and free of HSM dependencies during development.**

[![Live Demo](https://img.shields.io/badge/Live%20Demo-cyberchef.jacobmarks.com-blue)](https://cyberchef.jacobmarks.com)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://github.com/J8k3/CyberChef/blob/master/LICENSE)
[![Operations](https://img.shields.io/badge/Payment%20Operations-40%2B-green)](#recipe-catalog)
[![APC Validated](https://img.shields.io/badge/Cross--validated-AWS%20Payment%20Cryptography-orange)](#apc-cross-validation)

---

## What This Is

A workflow library for payment cryptography built on [CyberChef](https://github.com/gchq/CyberChef). Every operation is a composable pipeline step — you see exactly what's happening at each stage, share a recipe link with a colleague, or reproduce a result without access to an HSM.

Covers EMV (ARQC/ARPC, issuer-script MAC, PIN change), PIN blocks (formats, encrypted translation, IBM 3624, VISA PVV), MAC algorithms (AES-CMAC, ISO 9797, DUKPT, AS2805), DUKPT key derivation (TDES and AES), card validation data (CVV/CVV2/iCVV), key management utilities, TR-31/TR-34 parsing, and HSM command inspection (Thales payShield, Futurex Excrypt).

> **Not production tooling.** This is for development, testing, debugging, and interoperability work. Operations use clear keys in software. Do not use with production cryptographic keys, real PANs, or live PIN blocks.

---

## Live Demo

**[cyberchef.jacobmarks.com](https://cyberchef.jacobmarks.com)** — all payment operations are available under the **Payments** category. Click any recipe link in this document to open it directly in the live tool.

<!-- screenshot placeholder -->
<!-- ![UI overview](screenshots/cyberchef-payments-ui-overview.png) -->
<!-- *[Screenshot: CyberChef UI with Payments category expanded, showing operation groups]* -->

---

## Why Workflows, Not Just Operations

Every HSM vendor and cloud service implements the same cryptographic primitives. The value in payment cryptography work — especially during integration, migration, and debugging — is in the pipeline:

- **Inspectable intermediate state** — see the CDOL1 preimage before computing an ARQC, see the decrypted PIN block before re-keying it, see the exact MAC input before generating a MAC
- **Shareable recipes** — send a colleague a URL that opens the exact same operation with the same parameters pre-filled
- **Reproducible results** — validate your production implementation against a known-good software reference without a second HSM
- **Step-through debugging** — CyberChef's breakpoint feature lets you pause between operations and inspect what the data looks like at each stage

These properties make it practical to debug a MAC mismatch with a network processor, verify an ARQC implementation before going to the scheme, or walk a vendor through exactly what your system is computing.

---

## Recipe Catalog

### EMV

**ARQC / ARPC — Transaction Cryptogram Workflows**

EMV transaction authentication. Assemble the CDOL1 preimage from named fields, generate or verify the ARQC using the derived session key, then generate the issuer ARPC response.

| Recipe | Link |
|---|---|
| Generate ARQC | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Generate%20ARQC%20Workflow')EMV_Build_ARQC_Data('000000001000','000000000000','0840','0000000000','0840','260521','00','00000000','5900','0001','Hex')EMV_Generate_ARQC('00112233445566778899AABBCCDDEEFF',8,true)) |
| Generate then verify ARQC (full chain) | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Generate%20then%20verify%20ARQC%20(full%20chain)')Key_Generate('AES-128%20(16%20bytes)',16,true,false)Register('(%5B%5C%5Cs%5C%5CS%5D*)',true,false,false)EMV_Build_ARQC_Data('000000001000','000000000000','0840','0000000000','0840','260521','00','A1B2C3D4','5900','0001','Hex')EMV_Generate_ARQC('$R0',8,false)EMV_Verify_ARQC('$R0',8,'000102030405060708090A0B0C0D0E0F',true)) |
| Generate ARPC issuer response | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Generate%20ARPC%20issuer%20response')Key_Generate('AES-128%20(16%20bytes)',16,true,false)Register('(%5B%5C%5Cs%5C%5CS%5D*)',true,false,false)EMV_Build_ARQC_Data('000000001000','000000000000','0840','0000000000','0840','260521','00','A1B2C3D4','5900','0001','Hex')EMV_Generate_ARQC('$R0',8,false)Register('(%5B%5C%5Cs%5C%5CS%5D*)',true,false,false)EMV_Build_ARPC_Data('Method%201%20(Visa/Amex/Discover)','$R1','5931','00000000','','Hex')EMV_Generate_ARPC('$R0',8,true)) |

Operations: `EMV Build ARQC Data`, `EMV Parse ARQC Data`, `EMV Generate ARQC`, `EMV Verify ARQC`, `EMV Generate ARPC`, `EMV Build ARPC Data`, `EMV Parse ARPC Data`, `EMV Parse TLV`

Notes:
- CDOL1 structure is network-agnostic — the same 10-field 33-byte layout applies across Visa, Mastercard, Amex, Discover, and JCB
- ARPC supports Method 1 (Visa/Amex/Discover) and Method 2 (Mastercard) — select the correct method in the args
- `EMV Parse TLV` decodes BER-TLV encoded EMV data (DE 55, ICC responses, GPO responses)
- APC cross-validation note: `verify_auth_request_cryptogram` requires AES-256 E0 keys; AES-128 is rejected by the service

<!-- screenshot placeholder -->
<!-- ![EMV ARQC workflow](screenshots/emv-arqc-workflow.png) -->
<!-- *[Screenshot: CDOL1 assembly → ARQC generation → ARQC verification, showing each step's output]* -->

---

**Issuer-Script MAC and PIN Change**

Build issuer-script APDUs, compute the session MAC, and prepare PIN change commands.

| Recipe | Link |
|---|---|
| Generate issuer-script MAC | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Generate%20issuer-script%20MAC')Key_Generate('AES-128%20(16%20bytes)',16,true,false)Register('(%5B%5C%5Cs%5C%5CS%5D*)',true,false,false)EMV_Build_Script_Data('84','PUT%20DATA','00','42','0102030405060708090A','Hex')EMV_Generate_MAC('$R0','Method%202',8,true)) |
| Verify issuer-script MAC | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Verify%20issuer-script%20MAC')EMV_Verify_MAC('0123456789ABCDEFFEDCBA9876543210','22CB48394DFD1977','Method%202',true)&input=ODQyNDAwMDAwODk5OUU1N0ZEMEY0N0NBQ0UwMDA3) |

Operations: `EMV Build Script Data`, `EMV Build PIN Change Script Data`, `EMV Generate MAC`, `EMV Verify MAC`, `EMV Generate MAC (PIN Change)`

Notes:
- Padding method selector: Method 2 (ISO 7816-4, default for EMV issuer scripts) vs Method 1 (zero-pad only, required by some host implementations). Both sides of a generate/verify pair must use the same method.
- PIN change flow: `EMV Build PIN Change Script Data` assembles the 5-byte `84 24 P1 P2 Lc` header; the encrypted PIN block is appended by `EMV Generate MAC (PIN Change)` before computing the MAC

---

### PIN

**PIN Block Formats — Build, Parse, Translate**

| Recipe | Link |
|---|---|
| Build ISO Format 0 then parse (full chain) | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Build%20ISO%20Format%200%20PIN%20block%20then%20parse')PIN_Generate(4,'PIN%20digits','')PIN_Block_Build('ISO%20Format%200','5432101234567890',false)PIN_Block_Parse('ISO%20Format%200','5432101234567890')) |
| Re-key encrypted PIN block between ZPKs | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Re-key%20encrypted%20PIN%20block%20between%20ZPKs')PIN_Block_Translate_Encrypted('DDDDEEEEFFFFAAAABBBBCCCCDDDDEEEE','ISO%20Format%200','5432101234567890','AABBCCDDEEFF00112233445566778899','ISO%20Format%200','5432101234567890',false)&input=N0YzODFEQkY5RjY5MDZDNA) |
| Re-key with JSON inspection output | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Re-key%20encrypted%20PIN%20block%20(JSON%20inspection%20output)')PIN_Block_Translate_Encrypted('DDDDEEEEFFFFAAAABBBBCCCCDDDDEEEE','ISO%20Format%200','5432101234567890','AABBCCDDEEFF00112233445566778899','ISO%20Format%200','5432101234567890',true)&input=N0YzODFEQkY5RjY5MDZDNA) |

Operations: `PIN Block Build`, `PIN Block Parse`, `PIN Block Translate`, `PIN Block Translate Encrypted`, `PIN Data Generate`, `PIN Data Verify`

`PIN Block Translate Encrypted` is the acquirer's core PIN routing operation: decrypt an encrypted PIN block under an incoming zone key (ZPK/PEK), optionally change format, and re-encrypt under an outgoing zone key.

Supported formats: ISO 9564 Format 0, Format 1, Format 3. `PIN Block Translate Encrypted` uses TDES-ECB; accepts 2-key (16-byte) or 3-key (24-byte) keys.

<!-- screenshot placeholder -->
<!-- ![PIN block translation](screenshots/pin-block-translate-encrypted.png) -->
<!-- *[Screenshot: Encrypted PIN block re-keying between ZPKs with JSON inspection output showing decrypt → translate → re-encrypt]* -->

---

**PIN Verification — IBM 3624 and VISA PVV**

Issuer-side PIN verification artifact generation and verification.

| Recipe | Link |
|---|---|
| VISA PVV: generate from clear PIN | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('VISA%20PVV%20generate%20from%20clear%20PIN')PIN_Generate(4,'PIN%20digits','')VISA_PVV_Generate('0123456789ABCDEFFEDCBA9876543210','5432101234567890',1,true)) |
| VISA PVV: generate then verify (full chain) | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('VISA%20PVV%20generate%20then%20verify')PIN_Generate(4,'PIN%20digits','')VISA_PVV_Generate('0123456789ABCDEFFEDCBA9876543210','5432101234567890',1,false)VISA_PVV_Verify('0123456789ABCDEFFEDCBA9876543210','5432101234567890',1,'1234',true)) |
| IBM 3624: generate PIN offset | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('IBM%203624%20generate%20PIN%20offset')PIN_Generate(4,'PIN%20digits','')PIN_IBM_3624_Offset_Generate('0123456789ABCDEFFEDCBA9876543210','0123456789012345','5432101234567890','F',false)) |
| IBM 3624: generate then verify (full chain) | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('IBM%203624%20generate%20then%20verify')PIN_Generate(4,'PIN%20digits','')PIN_IBM_3624_Offset_Generate('0123456789ABCDEFFEDCBA9876543210','0123456789012345','5432101234567890','F',false)PIN_IBM_3624_Verify('0123456789ABCDEFFEDCBA9876543210','0123456789012345','5432101234567890','F','1234',true)) |

Operations: `PIN IBM 3624 Offset Generate`, `PIN IBM 3624 Verify`, `VISA PVV Generate`, `VISA PVV Verify`

---

### MAC

**Message Authentication — AES-CMAC, ISO 9797, HMAC, DUKPT, AS2805**

| Recipe | Link |
|---|---|
| Generate AES-CMAC | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Generate%20AES-CMAC')MAC_Generate('Hex','AES-CMAC','00112233445566778899AABBCCDDEEFF','Hex','','Method%201',8,false)&input=MTEyMjMzNDQ1NTY2Nzc4OA) |
| Verify AES-CMAC | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Verify%20AES-CMAC')MAC_Verify('Hex','AES-CMAC','00112233445566778899AABBCCDDEEFF','Hex','','Method%201','339AF1AD1650E908',true)&input=MTEyMjMzNDQ1NTY2Nzc4OA) |

Operations: `MAC Generate`, `MAC Verify`

Supported algorithms: HMAC-SHA-224/256/384/512, AES-CMAC, TDES-CMAC, ISO 9797-1 Algorithm 1, ISO 9797-1 Algorithm 3, AS2805-4.1, DUKPT MAC Request CMAC, DUKPT MAC Response CMAC, DUKPT ISO 9797-1 Alg 1, DUKPT ISO 9797-1 Alg 3.

Note: ISO 9797-1 Algorithm 3 is Retail MAC (TDES outer CBC). DUKPT MAC methods expect a clear BDK plus full KSN. EMV MAC operations are in the EMV section above.

---

### DUKPT

**Derived Unique Key Per Transaction — TDES and AES**

| Recipe | Link |
|---|---|
| TDES DUKPT: derive IPEK from BDK | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('TDES%20DUKPT%20derive%20IPEK%20from%20BDK')Key_Generate('TDES%20Double-length%20(16%20bytes)',16,true,false)DUKPT_Derive_TDES_Key('Derive%20IPEK','FFFF9876543210E00008','None',false)) |
| TDES DUKPT: derive PIN session key | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('TDES%20DUKPT%20derive%20PIN%20session%20key')Key_Generate('TDES%20Double-length%20(16%20bytes)',16,true,false)DUKPT_Derive_TDES_Key('Derive%20Session%20Key','FFFF9876543210E00008','PIN',false)) |

Operations: `DUKPT Derive TDES Key`, `DUKPT Derive AES Key`

Notes:
- `DUKPT Derive TDES Key`: ANSI X9.24-1, 10-byte KSN, IPEK-based. Verified against published X9.24-1 test vector.
- `DUKPT Derive AES Key`: ANSI X9.24-3, 12-byte KSN, IK-based, AES-128. Verified against official X9.24-3 §6.3 test vectors.
- **DUKPT TDES data encryption variant note**: CyberChef follows the ANSI X9.24-1 "Data" variant (bytes 5+13 XOR `0xFF`). APC uses a different undocumented internal variant for data encryption. DUKPT MAC operations align correctly between the two.

<!-- screenshot placeholder -->
<!-- ![AES DUKPT derivation chain](screenshots/aes-dukpt-derivation.png) -->
<!-- *[Screenshot: AES DUKPT key derivation showing BDK → IK → transaction key → working key chain in JSON output]* -->

---

### Card Validation

**CVV, CVV2, iCVV, PAN**

| Recipe | Link |
|---|---|
| Generate CVV2 | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Generate%20CVV2')Card_Validation_Data_Generate('CVV2%20/%20CVC2%20(force%20000)','4123456789012345','02','25','MMYY','101',3,false)&input=MDEyMzQ1Njc4OUFCQ0RFRkZFRENCQTk4NzY1NDMyMTA) |
| Verify CVV2 | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Verify%20CVV2')Card_Validation_Data_Verify('CVV2%20/%20CVC2%20(force%20000)','4123456789012345','02','25','MMYY','101','221')&input=MDEyMzQ1Njc4OUFCQ0RFRkZFRENCQTk4NzY1NDMyMTA) |
| PAN Generate: Visa curated test card | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Generate%20Visa%20curated%20test%20PAN')PAN_Generate('Visa','Curated%20sample',16,'Any',true)) |
| PAN Parse: classify by network | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Generate%20Mastercard%20PAN%20and%20parse')PAN_Generate('Mastercard','Curated%20sample',16,'5-series%20(51-55)',false)PAN_Parse()) |

Operations: `Card Validation Data Generate`, `Card Validation Data Verify`, `PAN Generate`, `PAN Parse`

CVV2 forces service code `000`. iCVV forces service code `999`. CVV/CVC uses the service code you supply. `PAN Parse` outputs card type, confidence, major industry identifier, Luhn validity, and network.

Recommended chain for card test setup: `PAN Generate` → `PAN Parse` → `Card Validation Data Generate`

---

### Key Management

**Key Generation, KCV, Component Handling, ECDH, TR-31/TR-34**

| Recipe | Link |
|---|---|
| Generate key and compute AES-CMAC KCV | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Generate%20AES-128%20key%20and%20compute%20KCV')Key_Generate('AES-128%20(16%20bytes)',16,false,false)Payment_Calculate_KCV('Hex','AES-CMAC%20(Empty)',6)) |
| TR-31 key block: parse and inspect | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Parse%20TR-31%20key%20block%20header%20fields')TR-31_Parse_Key_Block(false)&input=RDAxMTJQMEFFMDBFMDAwMDEwRUY5OTkwQzgwMkMzRUM3REEwNEM2OUFENjhBNzFCMjM4ODBEQzZDQTY0QjY0Q0UyRTVGMUE0RDA5NTJBM0E) |

Operations: `Key Generate`, `Payment Calculate KCV`, `Key Component Split`, `Key Component Combine`, `Derive ECDH Key Material`, `AS2805 Generate KEK Validation`, `TR-31 Parse Key Block`, `TR-34 Parse Key Transport`

Notes:
- `Key Component Split` / `Key Component Combine` use XOR shares — all N components required to reconstruct the key (no threshold/Shamir scheme). For test use; production key ceremonies require a certified HSM.
- `TR-31 Parse Key Block` decodes all X9.143 header fields with descriptions and PCI compliance flags.
- `TR-34 Parse Key Transport` handles B0–B9 message types, error codes, and peeks at the outer ASN.1 SEQUENCE of the CMS envelope.
- ECDH chain: `Derive ECDH Key Material` → optional KDF → `AES Key Wrap` / `AES Key Unwrap`. Not a full TR-34 or APC `TranslateKeyMaterial` implementation by itself.

---

### HSM Command Inspection

**Thales payShield and Futurex Excrypt Triage**

| Recipe | Link |
|---|---|
| Parse Thales payShield command | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Parse%20Thales%20payShield%20command')HSM_Parse_Thales_Command()&input=SEVBREhFMDEyMzQ1Njc4OUFCQ0RFRjAwMTEyMjMzNDQ1NTY2NzclMDBUQUlM) |
| Parse Futurex Excrypt command | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Parse%20Futurex%20Excrypt%20command')HSM_Parse_Futurex_Command()&input=W0FPR01BQztGUzY7UlYwMDExMjIzMzQ0NTU2Njc3O10) |

Operations: `HSM Parse Thales Command`, `HSM Parse Futurex Command`

These focus on transport-syntax triage: command identification, header/trailer, delimiter splitting, tag/value structure. Use them to confirm what command family you are looking at before reaching for the semantic payment or EMV operations. Pair with the [APC MCP Server](#related-tools) for deeper analysis and APC migration mapping.

Coverage: Thales payShield 10K (two-char command codes, configured header length), Futurex Excrypt Enterprise SSP v.2 (bracket-delimited tag/value, `AO` field as command code).

<!-- screenshot placeholder -->
<!-- ![HSM command triage](screenshots/hsm-thales-command-parse.png) -->
<!-- *[Screenshot: Raw payShield host message parsed into header, command code, and field split]* -->

---

## Chaining Patterns

Full-flow recipes combining multiple operations. Open any link to see the live chain.

| Workflow | Operations | Link |
|---|---|---|
| Brand test card setup | `PAN Generate` → `PAN Parse` → `Card Validation Data Generate` → `PIN Data Generate` | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Brand%20test%20card%20setup')PAN_Generate('Visa','Curated%20sample',16,'Any',true)) |
| EMV ARQC → ARPC end-to-end | `Key Generate` → `EMV Build ARQC Data` → `EMV Generate ARQC` → `EMV Build ARPC Data` → `EMV Generate ARPC` | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('EMV%20ARQC%20to%20ARPC%20end-to-end')Key_Generate('AES-128%20(16%20bytes)',16,true,false)Register('(%5B%5C%5Cs%5C%5CS%5D*)',true,false,false)EMV_Build_ARQC_Data('000000001000','000000000000','0840','0000000000','0840','260521','00','A1B2C3D4','5900','0001','Hex')EMV_Generate_ARQC('$R0',8,false)Register('(%5B%5C%5Cs%5C%5CS%5D*)',true,false,false)EMV_Build_ARPC_Data('Method%201%20(Visa/Amex/Discover)','$R1','5931','00000000','','Hex')EMV_Generate_ARPC('$R0',8,true)) |
| EMV issuer-script MAC | `Key Generate` → `EMV Build Script Data` → `EMV Generate MAC` | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('EMV%20issuer-script%20MAC')Key_Generate('AES-128%20(16%20bytes)',16,true,false)Register('(%5B%5C%5Cs%5C%5CS%5D*)',true,false,false)EMV_Build_Script_Data('84','PUT%20DATA','00','42','0102030405060708090A','Hex')EMV_Generate_MAC('$R0','Method%202',8,true)) |
| Encrypted PIN re-keying between ZPKs | `PIN Block Translate Encrypted` with JSON inspection | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Encrypted%20PIN%20block%20re-keying%20between%20ZPKs')PIN_Block_Translate_Encrypted('DDDDEEEEFFFFAAAABBBBCCCCDDDDEEEE','ISO%20Format%200','5432101234567890','AABBCCDDEEFF00112233445566778899','ISO%20Format%200','5432101234567890',true)&input=N0YzODFEQkY5RjY5MDZDNA) |
| TDES DUKPT: BDK → IPEK → session key | `Key Generate` → `DUKPT Derive TDES Key` (IPEK) → `DUKPT Derive TDES Key` (session key) | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('TDES%20DUKPT%20BDK%20to%20IPEK%20to%20session%20key')Key_Generate('TDES%20Double-length%20(16%20bytes)',16,true,false)DUKPT_Derive_TDES_Key('Derive%20IPEK','FFFF9876543210E00008','None',false)) |
| IBM 3624 offset generate → verify | `PIN IBM 3624 Offset Generate` → `PIN IBM 3624 Verify` | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('IBM%203624%20generate%20then%20verify')PIN_Generate(4,'PIN%20digits','')PIN_IBM_3624_Offset_Generate('0123456789ABCDEFFEDCBA9876543210','0123456789012345','5432101234567890','F',false)PIN_IBM_3624_Verify('0123456789ABCDEFFEDCBA9876543210','0123456789012345','5432101234567890','F','1234',true)) |
| VISA PVV generate → verify | `VISA PVV Generate` → `VISA PVV Verify` | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('VISA%20PVV%20generate%20then%20verify')PIN_Generate(4,'PIN%20digits','')VISA_PVV_Generate('0123456789ABCDEFFEDCBA9876543210','5432101234567890',1,false)VISA_PVV_Verify('0123456789ABCDEFFEDCBA9876543210','5432101234567890',1,'1234',true)) |
| Key generate → KCV cross-check | `Key Generate` → `Payment Calculate KCV` | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('Generate%20key%20and%20compute%20KCV')Key_Generate('AES-128%20(16%20bytes)',16,false,false)Payment_Calculate_KCV('Hex','AES-CMAC%20(Empty)',6)) |
| HSM command triage → payment analysis | `HSM Parse Thales Command` → appropriate payment operation | [open →](https://cyberchef.jacobmarks.com/#recipe=Comment('HSM%20command%20triage')HSM_Parse_Thales_Command()&input=SEVBREhFMDEyMzQ1Njc4OUFCQ0RFRjAwMTEyMjMzNDQ1NTY2NzclMDBUQUlM) |

---

## APC Cross-Validation

All HSM-mimic operations were compared against [AWS Payment Cryptography (APC)](https://aws.amazon.com/payment-cryptography/) using fixed test vectors imported as APC managed keys (2026-05-19). Keys used for testing only and scheduled for deletion after.

| Operation | Status | Notes |
|---|---|---|
| Payment Encrypt / Decrypt (TDES ECB) | ✅ Match | |
| MAC Generate / Verify (ISO 9797-3 Alg 1) | ✅ Match | |
| MAC Generate / Verify (AES-CMAC) | ✅ Match | |
| Card Validation Data (CVV / CVV2) | ✅ Match | |
| VISA PVV Generate / Verify | ✅ Match | APC `generate_pin_data` blocked by compliance warning; cross-validated via `verify_pin_data` |
| IBM 3624 Offset Generate / Verify | ✅ Match | Cross-validated via APC `verify_pin_data` |
| DUKPT TDES Key Derivation | ✅ Verified | Matches published ANSI X9.24-1 test vector |
| DUKPT AES Key Derivation | ✅ Verified | Verified against ANSI X9.24-3 §6.3 official test vectors |
| DUKPT TDES MAC | ✅ Match | APC `DukptKeyVariant=REQUEST` aligns with CyberChef "MAC Request" |
| EMV Generate MAC | ⚠️ Method-dependent | Method 2 (default, EMV standard) differs from Method 1 (zero-pad, APC default). Select the matching method on both sides. |
| EMV Generate ARQC | ❌ Blocked by APC constraint | APC `verify_auth_request_cryptogram` requires AES-256 E0; AES-128 rejected. CyberChef AES-CMAC + Option A session-key derivation is standard-compliant. |
| DUKPT TDES Data Encryption | ❌ Variant mismatch | CyberChef: ANSI X9.24-1 "Data" variant. APC uses an undocumented internal variant. MAC operations align correctly. |
| Payment Re-Encrypt | ❌ APC key-mode constraint | D0 keys with `NoRestrictions` rejected by APC `re_encrypt_data`. APC constraint, not a CyberChef bug. |

**Key findings:**
- APC `mac_length` is in nibbles (hex digits), not bytes — pass `16` to get an 8-byte MAC
- EMV MAC padding: both generate and verify must use the same method (Method 1 or Method 2)
- EMV ARQC on APC requires an AES-256 E0 master key

---

## Validation Status

| Operation | Validation class | Primary source |
|---|---|---|
| `PIN Block Build / Parse / Translate` | Vendor-aligned | AWS `GeneratePinData`; ISO 9564 |
| `PIN Block Translate Encrypted` | Vendor-aligned | AWS `TranslatePinData`; ISO 9564; PCI PIN Req 3-3 |
| `Key Component Split / Combine` | Verified | XOR key split — standard PCI key ceremony primitive |
| `Payment Calculate KCV` | Verified | NIST SP 800-38B |
| `DUKPT Derive TDES Key` | Externally cross-checked | ANSI X9.24-1 |
| `DUKPT Derive AES Key` | Externally cross-checked | ANSI X9.24-3 §6.3 official test vectors |
| `Derive ECDH Key Material` | Verified | RFC 3394; AWS `TranslateKeyMaterial` |
| `MAC Generate / Verify` (HMAC/CMAC) | Verified | NIST SP 800-38B |
| `MAC Generate / Verify` (ISO9797/DUKPT/AS2805) | Vendor-aligned | AWS MAC overview |
| `EMV Build / Parse ARQC Data` | Verified | CDOL1 per EMV Book 3 §10.1 |
| `EMV Build / Parse ARPC Data` | Verified | EMV Book 2 §8.2 |
| `EMV Parse TLV` | Verified | ISO 8825-1 BER-TLV; EMV Books 1–4 |
| `EMV Generate / Verify ARQC` | Vendor-aligned | AWS `VerifyAuthRequestCryptogram` |
| `EMV Generate ARPC` | Vendor-aligned | AWS `VerifyAuthRequestCryptogram` issuer flow |
| `EMV Generate / Verify MAC` | Vendor-aligned | AWS EMV MAC use case |
| `Card Validation Data Generate / Verify` | Vendor-aligned | AWS `GenerateCardValidationData` |
| `PIN IBM 3624 / VISA PVV` | Vendor-aligned | AWS IBM 3624 / VISA PIN verification objects |
| `PAN Generate` | Verified (Luhn/public ranges) | Discover public test-card page; Mastercard AVS scenarios |
| `PAN Parse` | Verified | Public card numbering rules |
| `TR-31 Parse Key Block` | Test helper | X9.143 header structure |
| `TR-34 Parse Key Transport` | Test helper | B0–B9 message types; ASN.1 CMS envelope |
| `HSM Parse Thales / Futurex Command` | Test helper | Thales payShield / Futurex Excrypt command syntax |

Validation classes:
- **Verified** — backed by a public standard plus deterministic local vectors
- **Vendor-aligned** — intentionally shaped to AWS Payment Cryptography or scheme/vendor semantics
- **Externally cross-checked** — checked against known-good vectors; governing spec not public
- **Test helper** — useful for testing, parsing, or workflow emulation; not a full standards-faithful implementation

---

## Related Tools

This repo is part of a small ecosystem of payment engineering tools:

| Tool | Role |
|---|---|
| [J8k3/CyberChef](https://github.com/J8k3/CyberChef) | Implementation — the fork where payment operations live |
| [J8k3/aws-payment-cryptography-mcp](https://github.com/J8k3/aws-payment-cryptography-mcp) | Cloud — AI-native MCP server for AWS Payment Cryptography with embedded payment domain knowledge |
| [J8k3/aws-payment-cryptography-hsm-proxy](https://github.com/J8k3/aws-payment-cryptography-hsm-proxy) | Migration — protocol translation layer between legacy HSM command sets and APC |
| [J8k3/Payments](https://github.com/J8k3/Payments) | Knowledge — shared payment standards reference consumed by the tools above |

---

## Non-Goals and Disclaimers

- Not a certified HSM, production key-custody platform, or PCI-scoped control surface
- Not an endorsement by or affiliation with AWS, Thales, Futurex, or any card scheme
- Not a replacement for scheme-certified testing tools or a QSA-equivalent review
- All operations use clear keys in software — there is no key custody, no tamper resistance, and no HSM boundary

Production payment systems require certified hardware security modules, proper key ceremonies, and PCI DSS compliance controls. These tools are for development, testing, debugging, and education.

---

## Standards References

- EMV Book 2 v4.4 — Issuer cryptography, ARQC/ARPC, secure messaging
- EMV Book 3 v4.4 — Application specification, CDOL1 layout
- ANSI X9.24-1 — TDES DUKPT
- ANSI X9.24-3 — AES DUKPT
- ISO 9564 — PIN management and security
- ISO 9797-1 — MAC algorithms
- ISO 8825-1 — BER-TLV encoding
- X9.143 / TR-31 — Interoperable secure key block
- TR-34 — Asymmetric key distribution
- NIST SP 800-38B — CMAC
- RFC 3394 — AES Key Wrap
- PCI PIN Security Requirements v3.1
