# Samsung VL350 native Spotify Connect investigation

**Report date:** 18 November 2026  
**Device under test:** Samsung VL350 / `WAH-VL350`  
**Reported firmware:** `VL350WWB-1005.1`  
**Platform reported by device:** Tizen 3.0 (`tizen30`)  
**Protocol version reported by device:** 2.3  
**Scope:** Determine why native Spotify Connect is not visible/usable, and assess whether firmware analysis can identify a way to restore it.

## Executive summary

The VL350 exposes a Spotify entry in its Samsung content-provider catalogue, but it has not been observed advertising the Spotify Connect mDNS service (`_spotify-connect._tcp`). This does **not** establish that Spotify Connect is unsupported, nor does the catalogue entry prove that the Spotify Connect receiver is installed, running or authenticated. The native Spotify issue remains unresolved.

The official Samsung firmware package for the VL350 is `WAM-HKA17WWC.zip`, version `1005.1`, listed as a USB-type upgrade file. Its extracted `upgrade.msd` is an `MSDU11(` package containing seven model headers and eight payload sections. The available unixtract MSD11 keys did not decrypt the model header. A total of 161 header/key test combinations produced no Tizen OUITH signature. This makes the bundled key set / current decryption assumptions insufficient for this package; it does not prove the package is corrupt.

The most useful next step is to establish whether the speaker's physical `SERVICE` connector exposes a usable service interface. The VL350 manual describes the SERVICE jack as being for firmware updates by a service engineer. The connector type and USB enumeration behaviour have **not** yet been established. Do not assume it is Micro-USB or connect it to a PC until the connector has been physically identified.

No firmware has been flashed, no factory reset has been performed, and no OCF security/ownership/ACL resources have been written.

## 1. Device identification

The active speaker's local information endpoint reported:

- Model: `WAH-VL350`
- Platform: `tizen30`
- Protocol version: `2.3`
- Firmware shown in SmartThings: `VL350WWB-1005.1`

The device exposed Samsung's local UIC/CPM service on TCP port `56001` during the observed session. Network listeners varied between scans and should not be treated as a fixed port map.

## 2. Spotify and content-provider findings

### 2.1 Samsung content-provider catalogue

A SmartThings phone-to-speaker session was captured on the local network. The exchange used a proprietary interactive AES-128 session (`2_AES_CBC` / `Connection: interactive`) rather than an ordinary cleartext HTTP session.

The decrypted CPM response contained a `CpList` catalogue. The captured response included these entries:

| Provider ID | Provider name | Reported `signinstatus` |
|---:|---|---:|
| 1 | Spotify | 0 |
| 2 | Deezer | 0 |
| 7 | BugsMusic | 0 |
| 19 | MelOn | 0 |
| 25 | TuneIn | 0 |

The response metadata reported `listtotalcount=25` and `listcount=25`, although only the five entries above were retained in the captured transcript. `signinstatus=0` was observed, but its exact semantics have not been independently established.

**Interpretation:** Spotify exists in the speaker's content-provider catalogue. That is evidence of a Spotify-related provider integration, not proof that the Spotify Connect receiver is present, enabled, signed in, functional, or currently supported by Spotify's service.

### 2.2 Spotify Connect discovery

`avahi-browse -rt _spotify-connect._tcp` did not show the VL350 during the observed scan. A third party smart speaker on the same LAN did appear as a positive control, confirming that the discovery command could see at least one Spotify Connect advertiser on the LAN.

**Interpretation:** The speaker was not observed advertising `_spotify-connect._tcp` at that time. This does not distinguish between a missing component, a disabled/stopped service, a startup/configuration fault, network-interface behaviour, or a service that only advertises in a particular state.

### 2.3 SSDP / UPnP

M-SEARCH probes were made for `ssdp:all`, root devices, MediaRenderer, AVTransport, ConnectionManager, RenderingControl and Samsung RemoteControlReceiver targets. No response from the active VL350 was observed during those tests.

The speaker did expose Samsung AllShare / UPnP-related resources on other ports during separate scans, including AVTransport and related service descriptions. The AVTransport description includes operations such as Play, Stop, Next, Previous, SetAVTransportURI, GetMediaInfo, GetTransportInfo and GetPositionInfo. No state-changing AVTransport command was used as part of this investigation.

### 2.4 SmartThings and local APIs

The speaker's UIC `MainInfo` response identified `WAH-VL350`, `tizen30` and protocol `2.3`. Feature information included `musiccontentsprovider=1`, `wifidlna=1` and `dmr=1`.

The `GetAvSourceAll` response contained only a virtual TV placeholder rather than a meaningful external source list.

Guessed paths such as `/cpath`, `/zeroconf` and `/spotifyConnect` returned 404 or empty responses. These path guesses are not evidence that the underlying services do not exist.

## 3. OCF / CoAP observations

The speaker exposed OCF CoAP-over-TCP/TLS endpoints during testing. Its TLS certificate identified it as a 2017 NetSpeaker OCF device. TLS 1.2 with ECDSA P-256 and `ECDHE-ECDSA-AES128-GCM-SHA256` was observed; the server sent a TLS `CertificateRequest` during tested handshakes.

An anonymous GET of `/oic/res` returned a CBOR resource directory, including OCF security resources and Samsung audio resources. Direct GET requests to Samsung audio resources such as `audioplayerFeature`, `feature`, `audioplayerErr`, `musicinfo`, `audioplayer`, `audioplayerMetadata` and `versionInfo` returned `4.01 Unauthorized`.

**Interpretation:** Discovery is possible, but the tested client did not possess the identity/authorization required to read the protected resources. The 4.01 responses do not prove that the resources are absent. No writes were attempted against `/oic/sec/doxm`, `/oic/sec/pstat`, `/oic/sec/acl2` or any speaker resource.

## 4. Debug interface observation

A TCP connect scan of port `26101` on the active speaker reported `closed` at the time of the scan. The local host did not have the `sdb` command available (`command -v sdb` returned no path).

**Interpretation:** No listener was detected on TCP/26101 during that scan. This does not rule out a disabled debug interface, a different port, a USB service connection, or a physical service interface.

## 5. Official firmware package and file integrity

Samsung's VL350 support page lists:

- File: `WAM-HKA17WWC.zip`
- Description: `Upgrade File (USB type)`
- Version: `1005.1`
- Listed modification date: 12 May 2018
- Listed size: approximately 405.31 MB

The package downloaded for analysis had these local measurements:

| File | Size | SHA-256 |
|---|---:|---|
| `WAM-HKA17WWC.zip` | 424,995,147 bytes | `e084891951a9deee44490282108d5f147ecc55a73dae57a0dc43c13a8ec93436` |
| extracted `image/upgrade.msd` | 424,926,033 bytes | `5a84740be407f8de1e7d49346547234378083df946e019c02802d21013f56340` |

The `.msd` file begins with `MSDU11(`. The unixtract parser identified eight sections and seven 3,608-byte model-header records. The selected header for the package name was `WAM-HKA17WWC`.

### 5.1 Header layout

Each of the seven headers has the same observed layout:

- 8-byte prefix beginning `fb4467ab` (the final four bytes differ per header)
- `Salted__` marker at byte offset 8 within the header record
- 8-byte salt
- 3,584-byte ciphertext region

The seven header record offsets in the `.msd` file were 361, 3,969, 7,577, 11,185, 14,793, 18,401 and 22,009. The `WAM-HKA17WWC` record had salt `a8d9f4d87296b159`.

### 5.2 Payload sections

The parser reported these sections (offsets are relative to the start of `upgrade.msd`):

| Section | Offset | Size (bytes) | Shannon entropy reported |
|---:|---:|---:|---:|
| 1 | 25,617 | 2,088,976 | 7.9999 bits/byte |
| 2 | 2,114,593 | 2,088,976 | 7.9999 bits/byte |
| 3 | 4,203,569 | 6,415,056 | 8.0000 bits/byte |
| 4 | 10,618,625 | 16 | 4.0000 bits/byte |
| 5 | 10,618,641 | 516,112 | 7.9996 bits/byte |
| 6 | 11,134,753 | 412,749,840 | 8.0000 bits/byte |
| 7 | 423,884,593 | 1,040,400 | 7.9998 bits/byte |
| 8 | 424,924,993 | 1,040 | 7.8049 bits/byte |

All eight section sizes are multiples of 16 bytes. `file` classified two sections as unrelated file types, but those detections are treated as false-positive magic matches because the data has near-maximum entropy. `binwalk` did not identify recognised embedded formats in sections 6 or 8.

**Interpretation:** The measurements are compatible with encrypted or otherwise high-entropy payloads. They do not, on their own, establish the cipher, encryption mode, or whether every section is encrypted. Section 4 is only 16 bytes long, so its entropy estimate is not statistically meaningful as a randomness test.

## 6. MSD11 decryption investigation

The local unixtract checkout loads its named key collections from `src/keys.ukf`. Its MSD11 collection contains 21 named entries, each with one key byte sequence. The MSD11 decoder selects the first byte sequence from each entry.

The original unixtract extraction attempt failed with:

```text
No matching key found!
```

A diagnostic test then tried the 21 MSD11 keys against all seven header records: 147 header/key combinations. It decrypted with no padding removal and searched the plaintext for the Tizen Software Upgrade Tree Binary Format v1.8 and v1.9 signatures anywhere in the output. No match was found.

A second test added two further 2015 AES-128 candidates (`T-HKM` and `T-HKP`) against all seven headers: 14 more combinations. No match was found.

**Total:** 161 tested header/key combinations, with no OUITH signature found.

The public `bugficks/msddecrypt` reference implementation is relevant because it documents the `MSDU11` container, searches each header for `Salted__`, and tests key candidates across headers. Its software AES path uses AES-128-CBC with the raw key bytes and `MD5(salt)` as the IV, without PKCS#7 unpadding. The reference project is described as a Samsung TV Tizen firmware decryptor; compatibility with the VL350's WAM package is not guaranteed. It also contains an optional TrustZone-backed SWU client path. The newer 32-byte AES-256 key derivation in unixtract is not validated by that reference's AES-128 software path.

**Interpretation:** The available key set and the tested derivations have not yielded a valid header. Likely explanations include a package/model-specific key not present in the tested collections, a different SWU/TEE passphrase arrangement, or an unrecognised format/derivation difference. The tests do not establish that the firmware is corrupt.

## 7. TEE / SWU lead

The reference `msddecrypt` source includes an optional TrustZone client using a TEE command named `CMD_SWU_PASSPHRASE`, and reads `/usr/share/org.tizen.tv.swu/itemsAESPassphraseEncrypted.txt`. Its software fallback performs AES-128-CBC as described above.

This is a useful lead, not a confirmed VL350 capability. The reference implementation targets Samsung Tizen TVs; no evidence has yet established that this exact speaker firmware contains the same TEE trusted application, service, encrypted passphrase file, or compatible interface. Confirming it would require legitimate read-only access to the speaker's own operating environment.

## 8. SERVICE connector and USB status

The VL350 manual labels a rear connector `SERVICE` and says it is used by a service engineer to update speaker firmware. Samsung's support page separately labels the downloadable firmware package as an `Upgrade File (USB type)`.

The SERVICE connector has **not** yet been tested from Linux. Its physical connector type, pinout, electrical interface and USB enumeration behaviour are unverified. The official wording does not establish that the SERVICE connector is a standard USB device port. Do not infer Micro-USB from the name `SERVICE`, and do not connect an unknown connector to a PC before identifying it physically.

## 9. What has not been done

- No firmware image has been modified, repacked or flashed.
- No factory reset has been performed.
- No changes have been made to OCF ownership, provisioning, security or ACL resources.
- No `SetAVTransportURI` or other disruptive playback command has been issued.
- No successful shell or SDB session has been established.
- The USB/service connector has not yet been connected to a host PC.
- No native Spotify Connect fix has been demonstrated.

## 10. Recommended next steps

1. **Identify the SERVICE connector physically before connecting it.** Compare the actual port shape and device revision with the manual/service documentation. The USB-A host/update path and the SERVICE connector should not be assumed to be the same interface.
2. **Perform a read-only host enumeration test once the connector and correct cable are identified.** Watch `dmesg`, then compare `lsusb` output before and after connection. Do not trigger an update or send unknown commands.
3. **If a documented USB serial or service interface enumerates, identify it without sending input first.** Capture descriptors and kernel driver binding; do not assume it is a shell.
4. **If read-only shell access becomes available, first inspect SWU/TEE presence and relevant files.** Do not edit security resources or system files.
5. **Repeat passive mDNS capture during a cold boot and source/mode changes.** This may show whether a Spotify service starts and then fails, or whether it never announces itself.
6. **Keep the native Spotify objective distinct from the SmartThings provider catalogue.** Finding a Spotify content-provider entry is not equivalent to restoring Spotify Connect.

## References

- [Samsung UK VL350 support page — firmware and manuals](https://www.samsung.com/uk/support/model/VL350/XE/). It lists `WAM-HKA17WWC.zip`, USB-type upgrade file version `1005.1`, and the VL350 manuals.
- [Samsung VL350 user manual, rear-panel SERVICE jack description](https://manualsdump.com/en/manuals/samsung-vl350-xe-vl550-xe-vl351-xe-vl551-xe/369169/10). Third-party manual mirror; the manual describes the SERVICE jack as for firmware updates by a service engineer.
- [bugficks/msddecrypt repository](https://github.com/bugficks/msddecrypt) — Samsung TV Tizen MSD firmware decryptor reference.
- [bugficks `msddecrypt.py`](https://raw.githubusercontent.com/bugficks/msddecrypt/master/src/msddecrypt.py) — header/key iteration and OUITH signature search.
- [bugficks `swu.py`](https://raw.githubusercontent.com/bugficks/msddecrypt/master/src/samygo/swu.py) — TrustZone client, `CMD_SWU_PASSPHRASE` and software AES fallback.

---

**Current conclusion:** The VL350's Spotify catalogue includes Spotify, but the speaker has not been observed advertising Spotify Connect. The native failure remains undiagnosed. Firmware extraction is blocked at the encrypted MSD11 header; the standard key candidates tested so far do not validate it. The next practical avenue is careful, read-only investigation of the SERVICE interface and, if access is obtained, the speaker's own SWU/TEE implementation.
