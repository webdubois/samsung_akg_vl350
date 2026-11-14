# Samsung VL350 Spotify Connect investigation

**Investigation period:** 13–14 November 2026  
**Device family:** Samsung WAH-VL350 / 2017 NetSpeaker  
**Observed firmware:** `VL350WWB-1005.1`  
**Platform / protocol reported by the speaker:** `tizen30` / `2.3`

## Executive summary

Two Samsung VL350 speakers were investigated, using ChatGPT, after they stopped appearing in Spotify's Connect device picker. Both speakers remain reachable on the LAN, respond to Samsung's local-control interface, and expose Samsung UPnP/ DLNA services. Their firmware also reports Spotify in its music-content-provider catalogue.

However, neither speaker was observed advertising the DNS-SD service `_spotify-connect._tcp.local`. During the same discovery test, the phone queried for that service and another brand's smart speaker on the network answered with a valid Spotify Connect service record. This is a useful positive control: the phone's discovery query and LAN mDNS visibility were working, but the VL350s did not advertise themselves as Spotify Connect devices.

The evidence therefore points to a **speaker-side Spotify Connect advertisement / ZeroConf problem**, or to the Spotify Connect component not being enabled or started. It does **not** yet identify the cause. In particular, the investigation has not established whether the provider is disabled, failing to initialise, unable to authenticate, or affected by a service-side change.

The firmware's Spotify provider catalogue entry and the Spotify Connect ZeroConf service are separate things. A catalogue entry is not proof that the Connect listener or its mDNS advertisement is running.

## Main findings

### 1. Device identity and firmware

The speakers identify as:

- Model: `WAH-VL350`
- Platform: `tizen30`
- Protocol version: `2.3`
- Software version: `VL350WWB-1005.1`

The two units were on the same LAN and were reachable at the time of testing.

### 2. Spotify appears in the Samsung content-provider catalogue

A SmartThings-to-speaker capture on TCP port `56001` exposed the speaker's Samsung interactive AES128-style session. The exchange began with a proprietary request beginning `2_AES_CBC`, followed by a speaker response containing `HTTP/1.1 211 AES128`. Subsequent requests used opaque/Base64-looking request paths and included headers such as `Connection: interactive` and `Accept-Encoding: aes128`.

Within that session, the speaker returned a `CPM` response for `CpList`. The captured XML included:

```xml
<method>CpList</method>
<response result="ok">
  <listtotalcount>25</listtotalcount>
  <liststartindex>0</liststartindex>
  <listcount>25</listcount>
  <cplist>
    <cp>
      <cpid>1</cpid>
      <cpname>Spotify</cpname>
      <signinstatus>0</signinstatus>
    </cp>
    <cp>
      <cpid>2</cpid>
      <cpname>Deezer</cpname>
      <signinstatus>0</signinstatus>
    </cp>
    <cp>
      <cpid>7</cpid>
      <cpname>BugsMusic</cpname>
      <signinstatus>0</signinstatus>
    </cp>
    <cp>
      <cpid>19</cpid>
      <cpname>MelOn</cpname>
      <signinstatus>0</signinstatus>
    </cp>
    <cp>
      <cpid>25</cpid>
      <cpname>TuneIn</cpname>
      <signinstatus>0</signinstatus>
    </cp>
  </cplist>
</response>
```

**Important qualification:** the response reported `listtotalcount=25` and `listcount=25`, but the excerpt recovered from the capture showed only five provider elements. Do not treat the five visible entries as a complete list of all 25 providers without obtaining the full response. The meaning of `signinstatus=0` has not been established; it should not automatically be interpreted as “signed out”.

A separate `Feature` response included:

```xml
<musiccontentsprovider>1</musiccontentsprovider>
<wifidlna>1</wifidlna>
<dmr>1</dmr>
```

This confirms that the speaker reports music-content-provider and DLNA/DMR features as enabled. It does not prove that Spotify's Connect ZeroConf server is active.

Other SmartThings responses identified the model, software version, speaker name, volume, mute state, current function, and platform. These were ordinary state/configuration responses; the current function varied with the speaker's state (for example, Bluetooth versus Wi-Fi/DLNA).

### 3. Spotify Connect mDNS discovery is missing from both VL350s

The phone's Spotify device-picker search generated DNS-SD queries for:

```text
_spotify-connect._tcp.local
```

The queries were visible over IPv4 and IPv6. Another brand's more modern smart speaker on the same LAN answered with a Spotify Connect service record. The observed third party record included:

- Service type: `_spotify-connect._tcp.local`
- TCP port: `4070`
- TXT record: `CPath=/spotifyConnect`

Neither VL350 advertised `_spotify-connect._tcp.local` in the captures. The broad discovery capture also showed no Spotify Connect service response from either speaker.

A separate capture of UDP port `5353` showed mDNS traffic from other LAN devices, confirming that the router-side capture was receiving multicast traffic. No mDNS packets sourced by either VL350 were observed in the relevant captures, including captures around a speaker restart.

This is strong evidence that the VL350s were not advertising Spotify Connect at the time of testing. It is not proof that every mDNS function or process on the speakers is permanently absent; advertisements can be conditional on state, role, or application startup.

### 4. The VL350s are active on the network and use Samsung discovery

The absence of Spotify mDNS does not mean that the speakers' network or discovery stacks are generally inactive.

A packet capture of one speaker showed repeated UDP payloads of this form:

```text
SEARCH BSDP/0.1
DEVICE=4
SERVICE=1
```

The packets were sent approximately every two seconds to:

- `239.255.255.250:15600`
- the LAN subnet broadcast address on UDP port `15600`

This traffic is consistent with Samsung-specific discovery traffic, often described as Samsung Service Discovery Protocol (BSDP/SDP) in community dissector material. It is not Spotify Connect discovery.

The speaker also used standard SSDP/UPnP on UDP port `1900`. During a phone discovery exchange, a VL350 sent an SSDP-style `HTTP/1.1 200 OK` response advertising its Samsung RemoteControlReceiver endpoint. The response identified:

```text
LOCATION: http://<speaker-ip>:7676/rcr/
ST: urn:samsung.com:device:RemoteControlReceiver:1
SERVER: SHP, UPnP/1.0, Samsung UPnP SDK/1.0
```

Thus, Samsung discovery and the RCR/UPnP service were working even though Spotify Connect was not being advertised.

### 5. Spotify Connect's usual ZeroConf mechanism

Spotify's commercial-hardware documentation describes a ZeroConf discovery mechanism. A compatible device advertises `_spotify-connect._tcp` using mDNS/DNS-SD, including an SRV record with the HTTP service port and a TXT record containing `CPath`.

The Spotify app then requests the ZeroConf API, typically with an HTTP `getInfo` request. A device that does not advertise the service will normally not be discovered through this path.

This is consistent with the observations in this investigation: the phone queried for the service, the third party (non-Samsung) device answered, and the VL350s did not.

Reference: [Spotify ZeroConf API documentation](https://developer.spotify.com/documentation/commercial-hardware/implementation/guides/zeroconf) and [Spotify ZeroConf FAQ](https://developer.spotify.com/documentation/commercial-hardware/implementation/faqs).

### 6. TCP listeners and UPnP service descriptions

A full TCP connect scan found several listeners. The port scan is a snapshot, and some ports appeared inconsistently between earlier and later probes (likely because the speakers were in differing wake states / input selected states at different times) ; ports should be rechecked before relying on them as a stable inventory.

The most useful confirmed listeners were:

| Port | Observed service / response | Notes |
|---:|---|---|
| `7676/tcp` | Samsung RemoteControlReceiver (RCR) | `GET /rcr/` returns a UPnP device description. |
| `8080/tcp` | Samsung WebServer | `/` returned 404; `/ws/app/` returned 503. Neither result identifies all valid application routes. |
| `9197/tcp` | UPnP Digital Media Renderer (DMR) | `GET /dmr` returns a UPnP device description. |
| `15500/tcp` | Open, unknown | Function not identified. |
| `56000/tcp` | Open, unknown | Function not identified. |
| `56001/tcp` | Samsung local control interface | SmartThings interactive session and `/UIC` requests observed. |

The later all-port scan reported the following additional listeners in that particular scan:

- Speaker A: `8187`, `57012`, and `59762` (purpose not identified).
- Speaker B (in a different wake state at this moment): `43172` (purpose not identified).

The scan also showed a large number of filtered/no-response ports on Speaker B, so it should not be treated as a perfectly complete inventory for that unit. Earlier targeted scans identified TLS listeners on `35163` and `57522`, presenting a Samsung OCF device certificate. Those ports did not appear in the later all-port scan, so their availability may be intermittent, state-dependent, or affected by scan behaviour. Their exact application-level roles were not established. A normal HTTP GET after TLS negotiation caused a TLS decode error/close, consistent with these not being ordinary HTTPS endpoints.

The third-party device advertised TCP port `4070`; that is the third party device's own advertised port, not a universal Spotify Connect port. A direct scan of port `4070` found it closed on both VL350s. The later all-port scan did not identify a Spotify Connect listener on either device. Because a Spotify implementation could use another port, the closed `4070` result alone does not prove that no ZeroConf HTTP service exists anywhere.

### 7. UPnP descriptions are the same on both speakers

The two speakers returned substantially identical UPnP device descriptions, with different friendly names and UUIDs.

#### RCR at `http://<speaker-ip>:7676/rcr/`

The description identifies:

```text
urn:samsung.com:device:RemoteControlReceiver:1
```

It exposes Samsung's `MultiScreenService`, with the control path:

```text
/RCR/control/MultiScreenService
```

The SCPD lists one action: `SendKeyCode`.

The RCR device description includes an `Application-URL` pointing to:

```text
http://<speaker-ip>:8080/ws/app/
```

A direct GET to `/ws/app/` returned HTTP 503. This only establishes that the requested root path was unavailable; it does not establish that the application framework or every route under `/ws/app/` is unavailable.

#### DMR at `http://<speaker-ip>:9197/dmr`

The description identifies a standard UPnP MediaRenderer:

```text
urn:schemas-upnp-org:device:MediaRenderer:1
```

It exposes four services:

- `RenderingControl`
- `ConnectionManager`
- `AVTransport`
- `StreamSplicing`

Both speakers published matching SCPD action lists.

The `AVTransport` service includes ordinary renderer operations such as `Play`, `Pause`, `Stop`, `Seek`, `SetAVTransportURI`, `GetMediaInfo`, and `GetTransportInfo`. Samsung-specific extensions include `X_PrefetchURI`, `X_DLNA_GetBytePositionInfo`, `X_GetStoppedReason`, and `X_PlayerAppHint`.

`X_PlayerAppHint` takes `InstanceID`, `UpnpClass`, and `PlayerHint`. Its existence does not establish that it launches a music provider or selects Spotify.

`StreamSplicing` exposes only:

- `SetBreakAuxStreamPlaylist(BreakId, BreakSpliceOutPosition, ExpirationTime, AuxStreamPlaylist)`
- `SetBreakAuxStreamTrigger(BreakId, BreakTriggerHigh, BreakTriggerLow)`

These are auxiliary stream-break/playlist operations, not identified Spotify controls. No Spotify-specific action was found in the published UPnP SCPDs.

The `RenderingControl` service exposes volume and mute operations alongside a number of video/TV-oriented extension actions. `ConnectionManager` exposes standard connection and protocol-information operations.

### 8. Samsung's CPM provider API: promising but not yet exercised correctly

A community reverse-engineered Samsung WAM API reference documents commands including:

- `GetCpList`
- `GetCpInfo`
- `SetCpService`

The older reference uses HTTP GET requests to `/CPM` on port `55001`. It includes examples for querying Spotify and selecting a provider by `cpservice_id`.

On the VL350, port `55001` was closed. Plain HTTP attempts to issue `GetCpList` and `GetCpInfo` through port `56001` did not return the requested method; instead, the server returned generic/session-dependent responses such as `CurrentFunc` or `MuteStatus`. This is not evidence that Spotify or CPM is absent. The actual SmartThings exchange uses a proprietary interactive AES128-style session, so the old cleartext WAM examples cannot be assumed to work unchanged.

The captured SmartThings session did successfully return `CpList`, including Spotify. A good next research task is to understand the VL350's interactive session well enough to issue a read-only `GetCpInfo` for Spotify and interpret the response. Do not try `SetCpService` until its effect and correct protocol on this model are understood; it may change playback/source state.

Reference: [Samsung WAM API method reference (community reverse-engineered)](https://github.com/bacl/WAM_API_DOC/blob/master/API_Methods.md). This documentation is for older WAM firmware and is not a guarantee of VL350 compatibility.

### 9. Other network and cloud observations

A 51-second capture during a device-picker test showed the speaker making DNS lookups for `connect.deezer.com` and `opml.radiotime.com`. It made an HTTP request to TuneIn's `opml.radiotime.com` endpoint. No identifiable Spotify DNS lookup or new Spotify connection from the speaker was observed in that short capture.

That absence is only a clue. The Spotify provider may not have been active, may use an already-open connection, may resolve through a cached result, or may only contact services after explicit selection.

The speaker was also observed communicating with Samsung cloud endpoints, including hostnames under `samsungiotcloud.com`, `samsungcloudsolution.com`, and `samsungqbe.com`.

A separate TLS capture showed these certificate chains:

| Hostname pattern | Certificate / issuer observed |
|---|---|
| `*.samsungcloudsolution.com` | Samsung Hubsite CA |
| `*.samsungiotcloud.com` | Samsung Electronics OCF Server SubCA, chaining to Samsung OCF Root CA |
| `*.samsungqbe.com` | Thawte TLS RSA CA G1, chaining to DigiCert Global Root G2 |

No eMudhra `emSign ECC Root CA - G3` or affected intermediate was seen in the TLS handshakes captured. This does not establish that the speaker's entire trust store lacks eMudhra certificates, or that every remote endpoint has been inspected. The local OCF certificate observed on the speaker is Samsung-issued, not eMudhra-issued. The OCF connection appeared to use certificate-based client/server authentication, making a generic HTTPS MITM proxy unsuitable without understanding that authentication.

**Conclusion on the eMudhra angle:** the captured chains provide no evidence that the eMudhra certificate incident is relevant to the observed VL350 connections. No attempt was made to impersonate a public service using compromised CA material.

### 10. OCF / CoAP probes

The speakers' OCF TLS certificate identified the device as a 2017 NetSpeaker and chained to Samsung's OCF CA hierarchy.

The speaker was observed joining IPv6 multicast groups including `ff02::158`, `ff03::158`, and `ff05::158`, as well as IPv4 multicast group `224.0.1.187`.

However, direct CoAP GET probes to `/oic/res` on UDP port `5683` (IPv4 and IPv6 link-local) did not receive responses. A DTLS probe to UDP port `5684` also received no response. These probes are inconclusive: they do not prove that OCF is absent, especially given the observed OCF TLS listeners and multicast membership.

## Reproducible checks

The following read-only checks were useful during the investigation. Replace `<speaker-ip>` with the speaker's current address.

### Check Spotify Connect mDNS

```bash
avahi-browse -rt _spotify-connect._tcp
```

A positive result should show an instance, host, port, and TXT data including `CPath`. The third party device provided a positive control in this investigation.

### Capture mDNS, SSDP, and Samsung discovery

On OpenWrt, with the LAN bridge name adjusted if necessary:

```bash
tcpdump -ni br-lan -e -nn -s0 -vv \
  'udp port 5353 or udp port 1900 or udp port 15600'
```

To write a capture file instead:

```bash
tcpdump -ni br-lan -s0 -w vl350-discovery.pcap \
  'udp port 5353 or udp port 1900 or udp port 15600'
```

When publishing captures, inspect and redact local addresses, MAC addresses, UUIDs, device identifiers, phone identifiers, user identifiers, and any credentials or tokens first.

### Inspect the UPnP device descriptions

```bash
curl -i --max-time 5 "http://<speaker-ip>:7676/rcr/"
curl -i --max-time 5 "http://<speaker-ip>:9197/dmr"
```

### Retrieve the SCPD action definitions

```bash
for path in AVTransport_1.xml ConnectionManager_1.xml \
            RenderingControl_1.xml StreamSplicing_1.xml; do
  echo "===== $path ====="
  curl -fsS "http://<speaker-ip>:9197/$path"
done
```

For Samsung's RCR service:

```bash
curl -fsS "http://<speaker-ip>:7676/rcr/MultiScreenService.xml"
```

### Check whether a port 4070 is open, as it was on the third party device:

```bash
nmap -n -Pn -sT -p 4070 <speaker-A-ip> <speaker-B-ip>
```

In this investigation, TCP 4070 was closed on both VL350s. This is not a definitive test for a ZeroConf server on another port.

## Firmware artifact notes

A firmware package named `WAM-HKA17WWC.zip` was examined locally. The archive and extracted `image/upgrade.msd` hashes were:

| Artifact | Size | SHA-256 |
|---|---:|---|
| `WAM-HKA17WWC.zip` | 424,995,147 bytes | `e084891951a9deee44490282108d5f147ecc55a73dae57a0dc43c13a8ec93436` |
| `image/upgrade.msd` | 424,926,033 bytes | `5a84740be407f8de1e7d49346547234378083df946e019c02802d21013f56340` |

The `.msd` file begins with `MSDU11(`. A `Salted__` marker was found at offset 369, followed by additional markers at regular 3,608-byte intervals through the first part of the file. The format and encryption/container structure have not been established, and the image has not been decrypted or modified. The package name and hashes are recorded here for reproducibility; this report does not claim that this package is installed on either speaker.

## Current assessment

### Confirmed by direct observation

- Both speakers identify as `WAH-VL350` and report software version `VL350WWB-1005.1`.
- SmartThings can communicate with the speakers through the Samsung local interface on TCP 56001.
- The firmware's `CpList` response includes Spotify (`cpid=1`).
- The speaker reports `musiccontentsprovider=1`.
- Both speakers expose Samsung RCR on TCP 7676 and a UPnP DMR on TCP 9197.
- Both DMR descriptions expose `RenderingControl`, `ConnectionManager`, `AVTransport`, and `StreamSplicing`.
- The speakers send Samsung-specific UDP discovery packets on port 15600 and participate in SSDP/UPnP discovery.
- The phone queries for `_spotify-connect._tcp.local`; the third party device responds; neither VL350 advertises the service in the captures.
- TCP 4070 is closed on both speakers.

### Not established

- Whether Spotify's provider executable/component is installed and starts successfully.
- Whether the provider is disabled, fails during startup, or fails while authenticating.
- Whether `signinstatus=0` has a specific signed-in/signed-out meaning on this firmware.
- Whether the speaker has a Spotify ZeroConf HTTP server on a port other than 4070.
- Whether the failure is caused by a firmware defect, a change in a Samsung service, a Spotify integration/backend change, or an account/provider-state issue.
- Whether the eMudhra incident affects any certificate store or endpoint not represented in the captured TLS handshakes.

### Working hypothesis

The most defensible current hypothesis is that **the Samsung music-provider catalogue still contains Spotify, but the Spotify Connect ZeroConf advertisement/service is not active on either VL350**. This is narrower and better supported than saying “Spotify has been removed” or “mDNS is broken”.

## Recommended next investigations

1. **Interrogate Spotify provider metadata read-only.** Work out the Samsung interactive session on TCP 56001 and obtain a genuine `GetCpInfo` response for `Spotify`. The older WAM cleartext examples are useful clues but are not directly compatible evidence for this VL350 firmware.
2. **Inspect firmware configuration and binaries.** Search the firmware contents for strings and configuration related to `Spotify`, `spotify-connect`, `zeroconf`, `Bonjour`, `_spotify-connect._tcp`, `CPath`, and CPM/provider startup. Identify the relevant process, init script, package, or feature flag before attempting changes.
3. **Look for an alternative ZeroConf listener.** The absence of service advertisement and closed TCP 4070 do not exclude another port. Search the firmware and, where possible, runtime process/listener tables rather than guessing ports.
4. **Compare the two speakers.** Their exposed UPnP service descriptions and reported firmware version appear equivalent. A controlled comparison of provider metadata, state, and boot-time logs could reveal whether both fail identically.
5. **Only then consider a controlled repair.** Avoid factory resets, unofficial firmware flashing, modifying trust stores, or attempting to impersonate Spotify/Samsung cloud services until the responsible component and recovery route are understood.

## References

- [Spotify for Developers — ZeroConf API](https://developer.spotify.com/documentation/commercial-hardware/implementation/guides/zeroconf) — official description of `_spotify-connect._tcp`, SRV/TXT records, `CPath`, and the HTTP ZeroConf API.
- [Spotify for Developers — ZeroConf FAQ](https://developer.spotify.com/documentation/commercial-hardware/implementation/faqs) — describes how a client discovers a device and requests `getInfo`.
- [Spotify for Developers — Technical requirements](https://developer.spotify.com/documentation/commercial-hardware/implementation/requirements/technical) — describes the need for a ZeroConf stack using mDNS and an HTTP server.
- [Samsung WAM API method reference](https://github.com/bacl/WAM_API_DOC/blob/master/API_Methods.md) — community reverse-engineered command reference for older WAM firmware, including `GetCpList`, `GetCpInfo`, and `SetCpService`; compatibility with VL350 is unverified.
- [Samsung WAM ports and services](https://sites.google.com/view/samsungwirelessaudiomultiroom/interfaces/ports-services) — community-maintained notes on Samsung soundbar RCR/DMR ports and advertised services; treat as a lead, not authoritative Samsung documentation.

## Privacy and publication notes

This report deliberately omits the speakers' real IP addresses, MAC addresses, UPnP UUIDs, serial-like identifiers, SmartThings mobile UUID, and user identifiers. Do not commit the original PCAPs, firmware binaries, device certificates with private material, or raw SmartThings captures to a public repository. Packet captures can contain persistent identifiers, hostnames, local network topology, and authentication-related data even when payloads appear opaque.

The firmware archive itself is not included. Only its filename, size, and SHA-256 hash are recorded.
