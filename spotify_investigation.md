# Samsung AKG VL350: Spotify Connect investigation update

**Update date:** 17 November 2026  
**Status:** Unresolved; investigation in progress  
**Scope:** Native Spotify Connect on Samsung AKG VL350 / WAH-VL350 speakers. This report concerns the original Samsung implementation, not third-party playback integrations.

## Summary

Two Samsung AKG VL350 speakers remain reachable on the local network and controllable through SmartThings. Native TuneIn playback works. However, neither speaker is currently discovered by Spotify as a Spotify Connect device, and the owner reports that Spotify and Deezer are no longer selectable in the SmartThings Music Services menu.

The firmware's Samsung content-provider (CPM) interface still lists Spotify and Deezer, and it accepts a request to select Spotify. That acknowledgement does **not** demonstrate that the Spotify application or Spotify Connect service has started. In the controlled test performed after TuneIn was stopped, the speaker remained in Bluetooth/no-device state rather than showing an active Spotify playback session. No `\_spotify-connect.\_tcp` advertisement was observed.

A separate SmartThings Community thread reports the same Spotify Connect disappearance on VL350 and VL550 speakers beginning in March 2026. Deezer Connect is a separate case: Deezer officially discontinued Deezer Connect for all devices in May 2025.

No fix has been established. The next technical step is offline analysis of the original firmware package to determine whether the Spotify Connect component, its launch configuration and its dependencies are still present.

## Device and firmware baseline

The tested units identify themselves as:

| Property | Observed value |
| - | - |
| Product/model string | `WAH-VL350` |
| Platform | `tizen30` |
| Samsung protocol version | `2.3` |
| Installed software version | `VL350WWB-1005.1` |
| Test speaker | `192.168.1.166` |
| Second speaker | `192.168.1.167` |


The IP addresses are local addresses from the test network and are included only to make the recorded tests reproducible. No account identifiers, device UUIDs or MAC addresses are included here.

Firmware files retained for analysis:

| File | Size | SHA-256 |
| - | -: | - |
| `WAM-HKA17WWC.zip` | 424,995,147 bytes | `e084891951a9deee44490282108d5f147ecc55a73dae57a0dc43c13a8ec93436` |
| `image/upgrade.msd` | 424,926,033 bytes | `5a84740be407f8de1e7d49346547234378083df946e019c02802d21013f56340` |


The `.msd` file begins with an `MSDU11(` marker and contains `Salted\_\_` byte sequences at offsets 369, 3977, 7585, 11193, 14801, 18409 and 22017. These observations suggest an encrypted or chunked container, but its structure and encryption scheme have **not** been established. No firmware has been modified or flashed.

## Current user-visible behaviour

- SmartThings can still discover/control the speakers.

- TuneIn is available through SmartThings and native TuneIn radio playback works.

- The owner reports that Spotify and Deezer have disappeared from the SmartThings Music Services menu. The exact date when each entry disappeared from the menu has not been established.

- Neither speaker appears in `avahi-browse -rt \_spotify-connect.\_tcp` during the tests. An unrelated third party smart speaker was visible in an earlier browse, providing a positive control for mDNS browsing on the LAN.

- DLNA functionality remains available. UPnP AVTransport, ConnectionManager and RenderingControl descriptions were retrieved from the speaker; this is consistent with the observed DLNA playback capability.

## Samsung local provider interface

The speakers expose a Samsung-specific UIC/CPM interface on TCP port `56001`. SmartThings traffic captures showed the phone communicating with the speakers on this interface. The interaction is not reliably reproduced by issuing isolated, stateless HTTP requests.

### Interactive request sequence

A repeatable way to issue a CPM request is to open a TCP connection to port `56001`, send a `GET /connect` request with `Connection: interactive`, then send the CPM request on the same socket. The `/connect` exchange produces empty HTTP 200 responses; the subsequent CPM command returns the XML result.

For example, the following request was accepted:

```
\<name\>SetCpService\</name\>  
\<p type="dec" name="cpservice\_id" val="1"/\>
```

The speaker returned both a `CpChanged` notification and a `SelectCpService` response containing:

```
\<response result="ok"\>  
    \<cpname\>Spotify\</cpname\>  
    \<timestamp\>2026-11-17T22:54:01Z\</timestamp\>  
    \<signinstatus\>0\</signinstatus\>  
\</response\>
```

This proves that the CPM command handler accepted the provider-selection request. It does **not** prove that Spotify playback began or that Spotify Connect was launched.

### Provider catalogue

CPM catalogue responses on both speakers included these entries:

| Provider | `cpid` |
| - | :-: |
| Spotify | `1` |
| Deezer | `2` |
| TuneIn | `25` |


Some responses reported `listtotalcount=25` and `listcount=25`, while the returned XML contained only a subset of entries. This report therefore records only the provider entries actually observed; it does not claim that the full 25-entry catalogue was recovered.

The fact that Spotify and Deezer remain in this local catalogue does not establish that their applications are currently launchable, that their cloud services remain compatible, or that SmartThings should display them.

### `GetCpInfo` comparison

Using the interactive connection, `GetCpInfo` returned:

- **Spotify:** `result="ok"`, `cpname=Spotify`, `signinstatus=0`, and a timestamp matching the preceding Spotify selection. No `audioinfo` block was returned.

- **TuneIn while playing:** `result="ok"`, `signinstatus=0`, and an `audioinfo` block containing the station title and `playstatus=play`.

- **TuneIn after it was deliberately stopped:** the same provider query returned `audioinfo` with `playstatus=stop`.

TuneIn therefore played successfully while reporting `signinstatus=0`. That value must not be interpreted as proof that Spotify is signed out, disabled or failing.

The Spotify timestamp matched the timestamp returned by the selection command rather than the time of the later query. It may be stored provider metadata; its exact meaning is unknown.

### Source mode and selection are not the same thing

`GetFunc` returned different states depending on the speaker's operating context:

- With the SmartThings Music Services screen active, it returned `function=wifi`, `submode=cp` in one set of checks.

- Outside that context, earlier checks returned `function=wifi`, `submode=dlna`.

- In a controlled test after TuneIn had been stopped, the speaker reported `function=bt`, `devicetype=none`, `connection=off` before and after `SetCpService` selected Spotify.

These observations show that `cp` is a generic content-provider mode, not proof of Spotify playback. They also show that `SetCpService` can return success without visibly transitioning the source to an active Spotify session.

During an earlier test, TuneIn continued playing after the programmatic Spotify selection. `GetCpInfo` for TuneIn still reported `playstatus=play`. This confirmed that the provider-selection acknowledgement had not replaced the active TuneIn stream.

A later controlled test stopped TuneIn first. The TuneIn provider then reported `playstatus=stop`; after Spotify selection the speaker remained in Bluetooth/no-device state. Spotify still did not appear in mDNS discovery.

### Interface caveats

- The owner observed that some UIC/CPM requests time out or return a generic UIC `MainInfo` response unless SmartThings is open on the Music Services screen.

- Isolated `curl` requests to `/CPM` are therefore not a dependable way to infer provider state. Use the interactive `/connect` exchange when reproducing CPM queries.

- A `GetPlayStatus` request to `/UIC` returned `CurrentFunc` on this firmware, not a distinct `PlayStatus` response. It has not been validated as a playback-status query for the VL350.

- `GetCpInfo` and `GetCpList` examples in the community-maintained [Samsung WAM API notes](https://github.com/bacl/WAM_API_DOC/blob/master/API_Methods.md) are useful protocol clues, but that document is based on older WAM products and is not authoritative VL350 documentation.

## Network observations

### Spotify Connect / mDNS

Repeated runs of:

```
avahi-browse -rt \_spotify-connect.\_tcp
```

returned no Spotify Connect records for either VL350. An unrelated third party smart speaker was visible during an earlier browse, so the browsing host was able to discover at least one `\_spotify-connect.\_tcp` service on the LAN.

### Packet captures

A capture was made on the LAN router’s interface while Spotify was selected through the interactive Samsung connection. The later capture included complete TCP and UDP packets for the test speaker over an approximately 68-second interval.

Within that interval:

- no `\_spotify-connect.\_tcp` mDNS announcement from the speaker was observed;

- no DNS query from the speaker was observed;

- no new outbound TCP connection from the speaker was observed after the selection request;

- Samsung local discovery broadcasts were observed, including `SEARCH BSDP/0.1 DEVICE=4 SERVICE=1` on UDP port `15600`.

These are bounded observations from one capture window. They do not prove the speaker never makes an external connection, nor do they rule out traffic before or after the capture.

An earlier, more restrictive capture retained UDP packets and TCP connection-control packets rather than full TCP payloads. It is not suitable for analysing application-level requests. Packet captures are not included in this report because they can disclose local network details and device identifiers; redact them before sharing publicly.

### UPnP and OCF observations

The speaker exposes Samsung AllShare / UPnP services, including AVTransport, ConnectionManager and RenderingControl descriptions. This corroborates that the DLNA renderer stack is present.

An OCF resource-directory request over CoAP-over-TCP/TLS to `/oic/res` returned a resource listing containing Samsung audio resources under `/sec/networkaudio/`, including player, playback, metadata, feature and volume resources. Direct GET requests to protected Samsung audio resources returned CoAP `4.01 Unauthorized`. This indicates that the resources exist but are not readable through the unauthenticated test client. No OCF ownership, provisioning or ACL resources have been modified. OCF authentication is not currently established as the cause of the Spotify problem.

## Related reports and service changes

### Similar VL350/VL550 Spotify Connect reports

A SmartThings Community thread opened on **25 March 2026** reports that Samsung AKG VL350 and VL550 speakers disappeared from Spotify Connect while remaining available in SmartThings and DLNA. The original poster reported that other Spotify Connect hardware on the same network remained discoverable and that router, speaker, SmartThings and Spotify-account troubleshooting had not restored the speakers. Another VL350 owner reported the same issue on 26 March 2026. Further users joined the thread in April.

On **4 April 2026**, the original poster wrote that a Samsung support representative had told them Samsung was receiving many reports and was looking into the issue; the poster also said Spotify had forwarded the case to its developers. These are statements by a community member about support conversations, not a published Samsung or Spotify service bulletin. The thread's **9 July 2026** update expressed the poster's view that the companies had abandoned the speakers, but it did not cite an official end-of-support announcement.

Source: [Samsung AKG VL350/550 speakers disappeared from Spotify Connect — SmartThings Community](https://community.smartthings.com/t/samsung-akg-vl350-550-speakers-disappeared-from-spotify-connect/308656)

### Deezer Connect

Deezer officially announced on **6 May 2025** that it was discontinuing Deezer Connect for all devices. That is a confirmed service-side change and is a plausible explanation for why Deezer Connect no longer works. It does not establish why the Deezer entry remains in the VL350's local provider catalogue, nor does it explain the separate Spotify Connect failure.

Source: [Say Goodbye to Deezer Connect — Deezer Community, 6 May 2025](https://en.deezercommunity.com/product-updates/say-goodbye-to-deezer-connect-80661)

## What is established, and what remains unknown

### Established by tests

1. The tested VL350 firmware identifies itself as `WAH-VL350`, platform `tizen30`, protocol `2.3`, software `VL350WWB-1005.1`.

2. The Samsung CPM catalogue includes Spotify (`cpid=1`), Deezer (`cpid=2`) and TuneIn (`cpid=25`) in the responses observed.

3. TuneIn playback works natively; its provider information reports station metadata and `playstatus=play` when playing.

4. The interactive CPM interface acknowledges `SetCpService` for Spotify and returns a successful `CpInfo` query.

5. Those successful responses do not prove Spotify playback or Spotify Connect startup. In the controlled no-TuneIn test, the speaker remained in Bluetooth/no-device state after Spotify selection.

6. The speakers did not advertise `\_spotify-connect.\_tcp` during the discovery tests.

7. The owner reports Spotify and Deezer are no longer available as selectable items in the SmartThings Music Services UI.

8. A separate multi-user report describes the same Spotify Connect disappearance on VL350/VL550 speakers in 2026.

### Not established

- Whether the Spotify Connect executable and its shared libraries remain installed in the firmware.

- Whether Spotify code is present but failing to start, disabled by configuration, or blocked by an obsolete/incompatible service endpoint.

- Whether Samsung's SmartThings provider list is driven by a cloud-side catalogue, local firmware configuration, or both.

- Whether Spotify has formally ended support for the VL350. No model-specific official discontinuation notice has been identified in this investigation.

- Whether there is a safe, technically viable way to restore native Spotify Connect.

The theory that the speaker is using obsolete Spotify endpoints or protocols is plausible but remains unverified. The absence of DNS or outbound TCP in a short capture window is not enough to prove it.

## Next investigation steps

1. **Analyse the firmware offline.** Determine the container/filesystem format of `upgrade.msd`; enumerate packages, executables, shared libraries, service definitions, startup scripts and configuration files.

2. **Search for Spotify components and configuration.** Look for Spotify/Connect-related binaries, process names, service launch definitions, libraries, certificate bundles, endpoint hostnames and discovery settings. Record paths and hashes; do not redistribute proprietary binaries or credentials.

3. **Establish whether a Spotify component is present.** Presence in the firmware is not enough: identify whether it is included in the image for this exact model and whether its startup entry is enabled.

4. **Compare working TuneIn and non-working Spotify paths.** Where possible, compare SmartThings commands and device-side events associated with a successful TuneIn start against Spotify selection. The current `SetCpService` command alone is not a launch command.

5. **Repeat only targeted, non-destructive tests on both speakers.** Preserve the current firmware and collect longer captures only when a specific startup/network hypothesis is being tested.

No factory reset, firmware flash, OCF ownership transfer, ACL write or security-state change has been performed. Those actions should remain off the table until the firmware and failure mechanism are understood.

## Questions for other VL350/VL550 owners

- Did Spotify Connect disappear suddenly, and if so, on what date?

- Did the Spotify entry disappear from SmartThings Music Services, or did it remain selectable while the Connect device disappeared from Spotify apps?

- Which exact model and firmware version are installed?

- Does native TuneIn still play?

- Do you see `\_spotify-connect.\_tcp` in an mDNS/Bonjour browser?

- Did the failure occur on multiple networks, including a phone hotspot?

Please report observations with exact dates and firmware versions. Avoid posting unredacted packet captures, account identifiers, device UUIDs, MAC addresses or credentials.

