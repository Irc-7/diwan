# Option C — Targeted Decryption: High-Value Token Inventory

Generated 2026-10-16 by `tools/scan_targets.py` (static IL scan of `inner-assembly.bin`).

## What this is

Every `ldc.i4 <token> ; call <decryptor>` call-site in the assembly was mapped to the
methods that also call **sink** APIs. The sink-adjacent tokens are the ones most
likely to decrypt into SQL queries, connection strings, URLs/endpoints, HTTP
headers, and AT-command/port config — the strings a Linux functional clone
actually needs.

## Numbers

- Decrypt call-sites found: **23,949** (matches the emulator's independent count exactly — validates the pattern scan)
- Sink memberrefs identified: 65 (`System.Data.SqlClient.*`, `System.Net.Http.*` / `System.Net.Web*`, `System.IO.Ports.SerialPort`)
- Sink-adjacent targets: **106 pairs → 100 unique tokens** (vs 11,421 unique tokens overall — a ~100x reduction)

| Sink family | Unique tokens in sink-touching methods | Sink-adjacent targets |
|---|---|---|
| SQL (SqlConnection/SqlCommand) | 259 | 24 |
| HTTP (HttpClient/WebRequest) | 178 | 74 |
| SerialPort | 6 | 2 |

## Highest-value targets (sample, verified against real IL)

- `0xF6629E15` → `WebRequest.set_ContentType` — confirmed in IL:
  `ldc.i4 -161309163 ; call decryptor ; callvirt set_ContentType`
- `0xF6629965` → `HttpWebRequest.set_UserAgent`
- `0xF6629E2A` → `WebRequest.get_Headers`
- 21× tokens → `HttpClient.get_DefaultRequestHeaders`, 16× → `HttpClient.PostAsync`
- 21× tokens → `SqlConnection.ClearAllPools` (connection-string management)
- 2× tokens → `SerialPort.set_Handshake` / `Dispose`

## How to use

`tools/targets.json` lists every target: method (`class::method`), token (signed +
hex), IL offset of the decrypt call, and the sink call it feeds. Decrypt any single
token via the caller-context approach (Option C) — the enumerator recursion that
stalled the bulk emulator is not needed when decrypting tokens individually.

## Files

- `tools/scan_targets.py` — the scanner (re-runnable)
- `tools/targets.json` — the full target list
