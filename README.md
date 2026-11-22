# APK Auditor

[![License: CC BY-NC-ND 4.0](https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-blue.svg)](LICENSE)
[![Version](https://img.shields.io/badge/Version-3.0-10D689.svg)](../../releases)
[![Pure JS](https://img.shields.io/badge/Pure-JavaScript-F7DF1E?logo=javascript&logoColor=black)](https://developer.mozilla.org/en-US/docs/Web/JavaScript)

Drop an APK on the page. The browser parses the DEX bytecode, decodes the binary AndroidManifest, reads the signing certificate, walks the components, hunts for secrets, and shows you a security score. Nothing leaves the tab.

Try it: [apkauditor.com](https://apkauditor.com)

![APK Auditor landing](docs/screenshots/landing-dark.png)

## What it actually does

It runs everything client-side. The APK ZIP is indexed with zip.js using a Blob-backed reader, then:

- **DEX parsing** parses in-memory DEX files up to a 256 MiB aggregate budget. Larger DEX files are staged in Origin Private File System (OPFS) and their string tables are read by bounded random-access slices; full bytecode decompilation for those large files is explicitly skipped with a warning. Up to 10 DEX files are considered.
- **AXML parser** decodes the binary AndroidManifest into XML so you can read it.
- **PKCS#7 parser** walks the signing certificate DER tree to extract subject, issuer, algorithm, validity, and fingerprint.
- **APK Signing Block scan** looks for the v2/v3 magic in the trailing bytes.
- **Resource decoder** reads compiled `resources.arsc` enough to flag hardcoded secrets in string resources.
- **Tracker detection** matches DEX strings against known SDK fingerprints (Firebase, AdMob, Crashlytics, Branch, and 30+ others).
- **Rule engine** applies 80+ static rules covering crypto, network, manifest, WebView, storage, and code patterns. Each finding is tagged with CWE, OWASP Mobile Top 10, and MASVS.
- **Entropy-aware secret detection** runs Shannon entropy over candidate strings to suppress noisy false positives. Text entries are decompressed into bounded chunks with a 64 KiB overlap so matches crossing chunk boundaries can still be found. Vendor regexes cover AWS, Stripe, GitHub, Slack, Google API keys, JWT, PEM blocks, Twilio, and more.

## Architecture (v3)

The 3300-line monolith is gone. Code is split into focused modules:

```
src/
  main.js              UI coordinator, tabs, rendering, drag and drop
  analyzer.worker.js   Web Worker entry point, isolates the analysis off the main thread
  styles.css           Theme, layout, components
  core/
    engine.js          AXML, DEX, PKCS#7, rule engine, tracker matching, scoring
    entropy.js         Shannon entropy and vendor secret patterns
    export.js          JSON / CSV / SARIF 2.1 export
    pdf.js             Printable PDF audit report
lib/
  zip.min.js
  zipjs.LICENSE.txt
  sha256.min.js         Incremental SHA-256
  md5.min.js            Incremental MD5
  jspdf.umd.min.js
```

Analysis runs in a Web Worker, so the UI stays responsive while an APK is being parsed. The selected `File` is passed directly to the worker. zip.js indexes the central directory through a Blob-backed random-access reader. SHA-256 and MD5 are updated incrementally from the File stream; the APK is no longer copied into one full-size hash buffer. Signing-block detection reads only the final 4 KiB. Entry extraction is bounded by declared and actual output size, with a 64 GiB aggregate expansion cap, a 1,000:1 entry compression-ratio guard, and per-format extraction limits.

Large DEX files (over the remaining 256 MiB aggregate in-memory DEX budget) are staged to OPFS, then their string-offset table and strings are read using random-access Blob slices. This keeps the full DEX off the JavaScript heap, but large-file class/method decompilation is currently skipped; the report includes a warning. OPFS is browser-managed storage and may be unavailable or quota-limited.

Text assets are decompressed through a WritableStream and scanned incrementally with a 64 KiB overlap; each entry is capped at 16 MiB of scan work and only a small preview is retained. The Explorer limits previews to 4 MiB and caps the rendered tree at 20,000 nodes. On browsers with the File System Access API, the Download action streams an entry directly to a user-selected file. The fallback download path is limited to 64 MiB. These safeguards make processing more memory-bounded, not a guarantee that every 16 GiB APK will fit browser CPU, storage, or runtime constraints. If the worker cannot be used, the page falls back to the main thread.

## Tabs

Six tabs cover the analysis:

- **Overview** - security score, app metadata, dangerous permissions, tracker SDKs
- **Findings** - all rule hits with severity filter, search, confidence slider, sort, expand-all, copy-match. Filter by severity (Issues / Info / Secure), confidence threshold, free-text search across rule name / file / match / CWE, and sort by severity, confidence, count, or name. Press `/` to focus search. Each instance has a copy-to-clipboard button.
- **Manifest** - parsed package info, permission table, raw decoded AndroidManifest.xml
- **Components** - activities, services, receivers, providers with exported flag and intent filters. Marks exported components in red and shows the intent filters that make them reachable from other apps. Implicit `exported=true` (when an intent filter is present but the attribute is omitted) is detected too.
- **Cert** - signing certificate subject, issuer, algorithm, validity, fingerprint, v1/v2 status. Flags debug certificates, expired certificates, and weak signature algorithms (MD5withRSA, SHA1withRSA). Tells you whether the APK has a v2/v3 signing block - APKs that ship only v1 are vulnerable to the Janus exploit on Android < 7.
- **Explorer** - file tree with viewer, hex dump for binaries, syntax-aware view for text. Opens AndroidManifest.xml, resources.arsc, any file inside `lib/`, `res/`, `assets/`, or `META-INF/`. Images render inline. Binaries get a paginated hex viewer. The Download button saves the current file to disk.

## Run it locally

Open `index.html` in a modern browser, or serve the directory:

```bash
python -m http.server 8765
# open http://localhost:8765/
```

There is no build step. No npm install. No service worker. It is plain JavaScript.

## Export

The Export menu emits four formats:

- **PDF** - printable audit with cover page, summary, findings (every instance), certificate, components, and trackers
- **JSON** - the full result tree
- **CSV** - flat findings table for spreadsheets
- **SARIF 2.1** - drop into GitHub Code Scanning

## Keyboard

| Key                | Action                          |
|--------------------|---------------------------------|
| `Alt + 1..6`       | Jump to tab                     |
| `/`                | Focus findings search           |
| `Esc`              | Collapse findings / close menus |
| `Arrow keys`       | Move between tabs               |

## Theme

Toggle button in the header cycles dark ↔ light. The choice is stored in localStorage. You can also force one with `?theme=dark` or `?theme=light` in the URL.

![Light theme](docs/screenshots/landing-light.png)

## Privacy

The selected APK stays on your device. Its `File` handle is passed to the Web Worker and read in bounded streams or slices; no APK bytes are uploaded. Large DEX files may be temporarily staged in browser-managed OPFS storage and removed after parsing. Analysis metadata and bounded previews are held in memory and discarded when you close the tab. There is no APK upload or telemetry. You can run it offline.

## Coverage

| Area      | Checks                                                                                              |
|-----------|-----------------------------------------------------------------------------------------------------|
| Manifest  | Debug flag, allowBackup, missing NSC, exported components, custom permissions, deep links           |
| Crypto    | MD5 / SHA-1 / DES / RC4, ECB mode, static IVs, hardcoded keys                                       |
| Network   | Cleartext HTTP, missing pinning, custom TrustManagers, hostname verifier bypass                     |
| WebView   | JavaScript enabled, file access, addJavascriptInterface, mixed content                              |
| Storage   | MODE_WORLD_READABLE / WRITABLE, external storage, SQLite raw queries, hardcoded SharedPrefs         |
| Code      | Reflection, dynamic class loading, runtime exec, clipboard, screenshot blocking                     |
| Secrets   | AWS / Stripe / Google / Firebase / GitHub / Slack / Twilio / JWT / PEM with entropy gating          |
| Signing   | v1-only, debug cert, expired cert, weak signature algorithm                                         |

## Limits

- APK/ZIP uploads are accepted up to **16 GiB**. This is a file-size ceiling, not a guarantee that a browser can process an archive of that size: whole-file hashing/signature inspection and DEX parsing still require large in-memory buffers. Actual limits depend on the browser, operating system, and available RAM.
- Each rule emits at most 20 instances per file, capped at 500 per rule globally
- Up to 5000 DEX classes are scanned for rule patterns (the parser still reads them all)
- Resource files larger than 300 KB are sampled rather than scanned end-to-end
- Hex viewer pages at 4 KB increments to keep large binaries responsive

## License

CC BY-NC-ND 4.0. See [LICENSE](LICENSE).

## Author

Built by [Sandeep Wawdane](https://github.com/thecybersandeep).

For authorized testing and educational use. Get permission before analyzing APKs you do not own.

## Sister tools

- [IPA Auditor](https://ipaauditor.com) - drag-drop iOS IPA static analyzer
- [ADB Auditor](https://adbauditor.com) - live Android audit over WebUSB and ADB
