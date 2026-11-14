# US-35 · The computer is at sea: vessels and buoys that have no API, call home when they can, and hold the only copy

| Field | Value |
|---|---|
| Primary persona | SRE |
| Secondary personas | ML, SEC, FIN, PE, DE |
| Workload class | edge computers with no provider API: 38 cargo vessels, each with an onboard server and six arm64 camera boxes managed by a fleet agent, reached by LEO satellite when available, L-band satellite at kilobits otherwise, and port Wi-Fi; 300 solar buoys with small arm64 boards that call home over Iridium twice a day; work that is pulled at contact, completes days later, and leaves its only copy on a disk at sea |
| Walkthrough | written against [ADR 0025](../adr/0025-a-computer-is-a-device-kubernetes-is-one-driver.md) decisions 2, 3, 4, 5 and 8 and its open item on computers at the edge with no provider API, [ADR 0029](../adr/0029-a-computers-file-systems-are-in-its-contract-staging-is-a-planned-copy.md) decisions 6 and 7 and its open item on edge computers holding the only copy, [ADR 0011](../adr/0011-device-tree-and-mount-table.md) decisions 2 and 7 and amendments 3 and 4, [ADR 0009](../adr/0009-stdout-artifact-stderr-log.md) amendment 1, [ADR 0004](../adr/0004-completion-contract-and-exit-codes.md) amendments 2 and 3, [ADR 0007](../adr/0007-secrets-are-references.md) amendment 4, [ADR 0008](../adr/0008-signal-routing.md), [ADR 0016](../adr/0016-identity-is-a-user-rights-belong-to-groups.md) decisions 4 and 11, [ADR 0024](../adr/0024-a-device-is-installed-by-terraform.md) decisions 6 and 8, [ADR 0026](../adr/0026-a-registry-is-a-device-a-tag-is-a-name-for-a-digest.md), US-10 and accepted limitation L10.8; it follows US-34 |
| Gaps already closed there | none |
| Milestones needed | M3, M4, M7, M8, M9, M10 |
| Findings in this document | L35.1 – L35.19 |

## 1. Story

**As an** SRE for a shipping company's fleet, **I want** the computers on our vessels and buoys to be computers in Convoy, so that a model update, a stop, a secret rotation or a question about where some footage is can be asked once, with an answer that says honestly which ships have heard it and when the rest will, **so that** a release is not "done" when 31 of 38 ships have it, footage is not "uploaded" while its only copy sits on a failing disk in the South Atlantic, an L-band link does not carry a 4 GB image at satellite prices, and a buoy that has not called home for nine days is not where the planner sends tomorrow's work.

**Context.** After US-34 the platform runs ADRs 0001 to 0030. US-10's sites are single-node k3s clusters with a link that drops; these computers are different.
- **Vessels.** Each of the 38 has an onboard x86 server with one GPU and six arm64 camera boxes with an NPU, run by AWS IoT Greengrass: the fleet's deployments are declared per group of devices in the cloud, and each device pulls its deployment when it connects. A vessel connects by LEO satellite broadband most days, by L-band satellite at kilobits when LEO is down, and by port Wi-Fi in port. Hull-inspection video and crew-area CCTV are recorded on the onboard server; CCTV is personal data under `residency=eu`.
- **Buoys.** 300 solar buoys run one container each on an arm64 board with 512 MB of memory, managed by the company's own MQTT agent. They call home over Iridium twice a day for a few minutes, and only run work when their battery is above 40 %.
- **Convoy.** ADR 0025 has no kind for these, and lists "computers at the edge with no provider API" as open. ADR 0029 lists "computers at the edge whose file system holds the only copy" as open.

## 2. Actors and identities

| Role | Who | Identity Convoy sees | Notes |
|---|---|---|---|
| Operates the fleet's computers | `soren` | user `soren`, group `marine-ops` | |
| Ships the hull model | `kenji` | user `kenji`, group `vision` | |
| Owns residency and device identity | `amara` | user `amara`, group `security` | |
| A vessel's master | the captain | none; the local control panel | may stop work on board |
| Crew | on board | none; local admin on the server | install their own containers |
| Another team | `telemetry` | group `telemetry` | deploys to vessels through the fleet manager's console |
| Pays | FinOps | group `finops` | satellite bytes |

## 3. Preconditions

- **Capabilities** (`caps`): on each vessel server: `gpu:1`, `arch:amd64`, links `leo` (metered per GB), `lband` (metered per MB), `port-wifi` (free); on camera boxes: `arch:arm64`, `npu`; on buoys: `arch:arm64`, memory 512 MB, link `iridium` (metered per KB), `power:solar`.
- **Packages**: none for these kinds; the fleet manager and the MQTT agent are run by `marine-ops` outside Convoy.
- **Platform defaults** `explain` must print: for work on a computer that connects by contact, when it was last heard from, when it is expected next, which link a transfer will use and at what price, when the work will be known to have run, and where its outputs are until they are delivered.

## 4. Main path

```bash
# Monday 08:00 · kenji, group vision
$ release hull-inspect --image=registry/vision/hull@sha256:9f2… --on /dev/compute/by-label/fleet=vessels
#  there is no computer kind for a vessel; soren had registered each server as `vm`,
#  so the plan tries to reach 38 APIs that do not exist                                 (L35.1)
#  the image is 4 GB; vessel 17 is on L-band this week; it pulls the image at satellite
#  prices; nothing priced it, and only two layers had changed                         (L35.6)
#  the camera boxes are arm64; the image has no arm64 variant; six boxes per ship download
#  it and fail                                                                            (L35.16)
#  the fleet manager's deployment for the vessel group, written by telemetry from the
#  console, is replaced by Convoy's, and their collector disappears from 38 ships       (L35.11)

# Wednesday · 31 vessels run v9; release says "done"                                      (L35.2)
#  5 more pick it up by Sunday; 2 have not called home since the release                 (L35.2)
#  vessel 22's master stops inspection during a storm from the onboard panel;
#  Convoy restarts it 40 minutes later                                                    (L35.12)
#  crew run their own video tool on vessel 9's GPU; Convoy's capacity says the GPU is free (L35.10)

# Thursday · soren: ls /sys/compute/by-label/fleet=vessels/
#  vessel 30 shows ready, 1 GPU free; its last contact was nine days ago               (L35.9)
#  a new analysis is placed on it                                                         (L35.9)
$ kill 8812     # a run on buoy 211, which is out of contact
#  exit 0; the buoy finishes the run at its next contact and reports success           (L35.7)

# Friday · vessel 14 docks in Santos; port Wi-Fi; the upload of 3 weeks of crew-area CCTV goes
#  to the nearest bucket, in sa-east-1                                                    (L35.4)
#  vessel 6's stage "hull-video-weekly" says succeeded; its outputs are on the onboard
#  disk, which fails before port; three weeks of hull video were never anywhere else (L35.3)

# The next weeks
#  vessel 25's onboard server is replaced in port; the new box is a new by-id (L10.8);
#  four processes on the old id stay pending for ever; the old disk, with the only copy
#  of 11 days of video, goes back to the office in a box                                (L35.8)
#  vessel 33 has been at sea for 46 days; its device certificate lived 30; it can fetch no
#  secret and pulls nothing                                                                (L35.5)
#  a camera box is stolen in port with a valid certificate                               (L35.5)
#  the registry's signing key is rotated; ships at sea hold the old trust root and refuse
#  every new image                                                                         (L35.17)
#  buoy receipts arrive with times up to 14 minutes off, some in the future             (L35.15)
#  buoys start a 30-minute stage at 38 % battery and stop at 20 %, 14 times             (L35.14)
#  the central nightly training waits on 38 vessels; its 24-hour deadline was inherited by
#  the onboard stages, which time out before most ships have called home               (L35.13)
#  vessel 2 never calls home again                                                        (L35.18)
#  footage recorded in a coastal state's waters, on a ship flagged elsewhere, is
#  "residency=eu" and nothing else                                                        (L35.19)
```

| Step | Convoy object | Fleet object | Where the state lives |
|---|---|---|---|
| a vessel's server | a `vm` computer, wrongly | a Greengrass core device | the fleet manager |
| a deployment | a release | a deployment for a group of devices | the fleet manager, pulled at contact |
| a buoy | none | an MQTT client | the company's agent |
| a buoy's work | none | a desired state in a topic | the broker, pulled at contact |
| outputs at sea | none | files on the onboard disk | the vessel, until port |
| device identity | none | X.509 certificates | the fleet manager's registry |

## 5. Abstraction coverage

| # | Aspect | Touched | How this story uses it | What to verify |
|---|---|---|---|---|
| A1 | Pipe: staged artifact, content id, cache | ○ | outputs that exist only on a device (L35.3) | |
| A2 | Command: class, annotation, executable fields | ○ | images for the device's architecture (L35.16) | AC-35.16 |
| A3 | Session: mount table, env, cwd, attach and detach | – | | |
| A4 | Backend selection | ● | placement on nine-day-old state (L35.9); power (L35.14) | AC-35.9, AC-35.14 |
| A5 | Completion contract | ● | a release over contacts (L35.2); `done-local` and `delivered` (L35.3); deadlines in contacts (L35.13) | AC-35.2, AC-35.3, AC-35.13 |
| A6 | Device tree | ● | no kind for computers without an API (L35.1); out of contact (L35.9) | AC-35.9 |
| A7 | Signals and lifecycle | ● | `kill` to a computer out of contact (L35.7); a stop by the master (L35.12) | AC-35.7, AC-35.12 |
| A8 | Secrets as references | ● | certificates that expire at sea; a stolen box (L35.5) | AC-35.5 |
| A9 | Cost | ● | satellite bytes (L35.6) | AC-35.6 |
| A10 | Identity and authorization | ● | device identity (L35.5); a local authority (L35.12) | AC-35.5 |
| A11 | Platform defaults and `caps` | ● | `contact_grace`, `contact_stale`, `contact_max`, link classes | AC-35.9 |
| A12 | Storage | ● | the only copy on an onboard disk (L35.3); a replaced server (L35.8) | AC-35.3, AC-35.8 |
| A13 | Networking | ● | LEO, L-band, Iridium and port Wi-Fi as priced link classes (L35.6) | AC-35.6 |
| A14 | Time and queueing | ● | deadlines counted in contacts (L35.13); device clocks (L35.15) | AC-35.13, AC-35.15 |
| A15 | Audit, receipts, retention and `replay` | ● | receipts with clock and location (L35.15, L35.19) | AC-35.15 |
| A16 | Durability | ● | the only copy (L35.3); a device that never returns (L35.18) | AC-35.3 |

## 6. Loophole probes

| # | Probe | Question for this story | Design answer | Finding |
|---|---|---|---|---|
| P1 | Imperative verb, declarative owner | the fleet manager's group deployments, also written from its console | ADR 0010 amendment 1: managed objects are intent | **L35.11** |
| P2 | POSIX semantics that cannot hold | `kill` to a process on a computer out of contact returns 0 | ADR 0011 amendment 4 queues commands to a disconnected device | **L35.7** |
| P3 | Orphaned or lost state | a replaced server's pending processes and its disk | L10.8: a replaced device is a new by-id | **L35.8** |
| P4 | Identity leak or escalation | a certificate that expires at sea; a stolen box | ADR 0007 amendment 4: site-local secret sources | **L35.5** |
| P5 | Invisible cost | a 4 GB image over L-band | L10.5: links have a price | **L35.6** |
| P6 | Lying cache | "ready, 1 GPU free" from nine days ago | ADR 0011 amendment 4: `stale=cache` with its age | **L35.9** |
| P7 | Silent failure | "succeeded" with the only copy on a disk that fails | ADR 0029 decision 7: a stage succeeds when its outputs arrive; not written for weeks at sea | **L35.3** |
| P8 | Partial fleet | a release that is "done" at 31 of 38 | ADR 0004 amendment 2's degraded channel, for partitions | **L35.2** |
| P9 | Created elsewhere | crew containers on the vessel's GPU | ADR 0024 decision 8: a device not installed by a module says so | **L35.10** |
| P10 | Multi-party action | a master stops work on board | ADR 0021 decision 10: people win | **L35.12** |
| P11 | Data at rest | crew CCTV uploaded to the nearest region | ADR 0011 amendment 3: residency on mounts | **L35.4** |
| P12 | Process-tree inheritance | a 24-hour deadline inherited by stages on ships that call home every few days | ADR 0004 amendment 3 | **L35.13** |

Seven more questions without a probe: is there a computer kind for devices with no API at all (**L35.1**, open); does placement know a buoy's battery (**L35.14**, rule); whose clock does a receipt use (**L35.15**, rule); does the plan check an image's architecture before shipping it (**L35.16**, rule); how does a device at sea verify images signed after it left (**L35.17**, rule); what can be promised about a device that never calls home again (**L35.18**, accepted); and does Convoy decide which law applies to data recorded at sea (**L35.19**, accepted).

## 7. Failure matrix

| Failure | Detected by | Exit | Receipt says | Recovery verb |
|---|---|---|---|---|
| devices not heard from by the deadline | the fan-out (L35.2) | 0 degraded at the floor, 1 below it | devices done, pending and silent | `release` again later, or `rm` the device |
| outputs not delivered | the stage (L35.3) | none until delivery; 1 at `contact_max` | `done-local`, the device, the bytes at risk | none; wait for contact or recover the disk |
| signal to a computer out of contact | `kill` (L35.7) | 0, `queued` | queued at, delivered at, exited at, which came first | none |
| device replaced | `replace` (L35.8) | none | pending work moved, outputs at risk on the old device | recover the old disk |
| certificate near expiry at sea | the fleet's renewal (L35.5) | none | the device, expiry, last contact | renew at contact |
| image without the device's architecture | plan (L35.16) | 3 | the image's platforms and the device's | build for the platform |
| stage started below the power threshold | the device (L35.14) | none; deferred | battery, threshold, forecast | none |

## 8. Acceptance criteria

- **AC-35.2** Given a fan-out to computers that connect by contact, then it is one process with a floor and a deadline; `ps -T` shows each device as `pending-contact`, `running`, `done-local`, `delivered` or `silent`; at the deadline it exits 0 at the floor (degraded below 100 %) or 1 below it, listing the devices not heard from.
- **AC-35.3** Given a stage on an edge computer, then its state is `done-local` until its outputs are delivered and verified at their destination, then `delivered`; the device never deletes outputs before delivery; `ls` of an output not yet delivered says `only copy on <device>`, and `explain` lists the bytes at risk per device.
- **AC-35.4** Given outputs with a residency, then an edge computer uploads them only to stores within that residency, whatever is nearest, and each output carries the device's reported location at the time it was written.
- **AC-35.5** Given an edge computer, then its identity is a device certificate whose lifetime exceeds its `contact_max` plus a margin and is renewed at contact; secrets for it are scoped to the device; revoking it takes effect at its next contact and is shown as `pending-contact` until then.
- **AC-35.6** Given a transfer to or from an edge computer, then it uses the link class its urgency allows (`--link=port|leo|any`), `explain` prices it per link class, and images move by changed layers only.
- **AC-35.7** Given a signal to a process on a computer out of contact, then the command returns at once with `queued until contact`; `wait` and the receipt record when the signal was delivered and when the process exited, and which came first.
- **AC-35.8** Given an edge computer replaced or removed, then `replace <old> <new>` carries its group, desired state and pending work to the new device, not its data; outputs left only on the old device are listed as at risk; processes on a device not heard from for `contact_max` become `lost`.
- **AC-35.9** Given a computer that connects by contact, then its state is `out-of-contact` between contacts with its last and expected next contact; `ls`, `ps` and `explain` print the age of what they show; placement uses state older than `contact_stale` only for work that can wait for the next contact; an alert is raised only when a contact is late by `contact_grace`.
- **AC-35.10** Given an edge computer, then the driver reads its full list of workloads at each contact; workloads Convoy did not deploy are shown as `outside Convoy` with their resource use, count against capacity, and are never removed by Convoy.
- **AC-35.11** Given an edge computer run by a fleet manager, then Convoy writes its intent as the fleet manager's deployment, merged with the components others declared, never replacing them; the fleet manager is the computer's operator.
- **AC-35.12** Given a stop by a local authority on the device (a master's panel, a site's safety switch), then the work shows `stopped locally` with the reason the device reports, and Convoy does not restart it until the device reports the stop lifted.
- **AC-35.13** Given a process tree whose children run on edge computers, then the children's deadlines count in contacts (`--contact-deadline=<n>` or a duration from first contact), bounded by the tree's deadline; the central stages proceed on what is delivered by the floor.
- **AC-35.14** Given a computer with a power budget, then it reports battery, threshold and forecast in `bounds`; a stage declares its energy with `--energy=<Wh>`; the device starts it only when the budget allows, and `ps` shows `waiting-power`.
- **AC-35.15** Given a receipt from an edge computer, then it carries `clock=synced|device`, the device's offset measured at the next contact, and times corrected by that offset, with the uncorrected value kept.
- **AC-35.16** Given an image and a set of devices, then the plan checks the image's platforms against each device's architecture before any transfer and refuses with exit 3, listing the devices it cannot run on.
- **AC-35.17** Given a trust root change, then the new root is delivered to edge computers before any image signed only with the new key is released to them, and `explain` lists devices that do not yet hold it.

## 9. Persona lenses

- **SRE**: wants a release to say which ships have it, which will, and which are silent, and wants no alert for a buoy that is merely between calls.
- **ML**: wants a model update that fits the link, the architecture and the battery, and a count of ships actually running it.
- **SEC**: wants crew footage in its residency whatever port the ship is in, device identities that survive a long voyage and die with a stolen box.
- **FIN**: wants satellite bytes priced before they are sent, and only changed layers sent.
- **PE**: wants the fleet manager to stay the operator, and crew workloads seen, not removed.
- **DE**: wants to know which data is still only on a ship, and how much.

## 10. Findings

| Id | Probe | Finding | Proposed disposition | Touches |
|---|---|---|---|---|
| L35.1 | none | Computers with no API are not in the design. ADR 0025's kinds all accept a submission; a vessel's server and a buoy accept nothing, and pull their work when they connect. There is no kind for them, no notion of contact, no state between "running" and "succeeded" for work whose result is on a disk at sea, and no answer to who holds the only copy. ADR 0025 and ADR 0029 leave it open. | **open** (drafted 2026-11-14 in [ADR 0031](../adr/0031-an-edge-computer-is-reached-by-contact-and-known-by-what-it-delivers.md): `edge` is a computer kind for devices that pull their desired state at contact, one computer per device and fleets as views; contact is in the contract and out of contact is a normal state; work moves through `pending-contact`, `running`, `done-local` and `delivered`; the only copy is stated and kept until delivered; transfers choose a link class and move only changed layers; signals arrive at contact and local stops win; device identity lasts the voyage; power and clock offsets are in the contract and the receipts; Convoy's own edge agent where there is no fleet manager). The record asked for: an `edge` computer kind whose driver publishes desired state that the device pulls at contact (through a fleet manager or Convoy's own agent); a `contact` field in the computer contract (last and expected next contact, link classes with their prices, power); process states `pending-contact`, `done-local`, `delivered`, `silent`, `lost`; fleets as views over many edge computers, never one computer; the only copy as a stated condition with its bytes at risk | ADR 0025 d2, d3, d5, d8; ADR 0011 a4; ADR 0009 a1; ADR 0029 d7 |
| L35.2 | P8 | A release to 38 vessels said "done" at 31; two were never heard from. | **rule** (applied: ADR 0031 rules; ADR 0004 rules; ADR 0025 rules; ADR 0026 rules; command map): a fan-out to computers that connect by contact is one process with a floor and a deadline; `ps -T` shows each device as `pending-contact`, `running`, `done-local`, `delivered` or `silent`; at the deadline it exits 0 at the floor (degraded below 100 %) or 1 below it, listing the devices not heard from | ADR 0004 a2; ADR 0025 d4; ADR 0026 |
| L35.3 | P7 | A stage "succeeded" with its outputs on an onboard disk that failed before port. | **rule** (applied: ADR 0031 rules; ADR 0029 rules; ADR 0006 rules; ADR 0009 rules; command map): a stage on an edge computer is `done-local` until its outputs are delivered and verified at their destination (ADR 0029 decision 7), then `delivered`; the device never deletes outputs before delivery; `ls` says `only copy on <device>` and `explain` lists the bytes at risk per device | ADR 0029 d7; ADR 0006; ADR 0009 a1 |
| L35.4 | P11 | Crew-area CCTV under `residency=eu` was uploaded to the nearest bucket, in sa-east-1. | **rule** (applied: ADR 0031 rules; ADR 0011 rules; ADR 0029 rules; ADR 0016 rules): an edge computer uploads outputs only to stores within their residency, whatever is nearest; each output carries the device's reported location when it was written | ADR 0011 a3; ADR 0029 d6; ADR 0016 d11 |
| L35.5 | P4 | A certificate expired after 30 of 46 days at sea, and a stolen camera box kept a valid one. | **rule** (applied: ADR 0031 rules; ADR 0007 rules; ADR 0016 rules; ADR 0024 rules): an edge computer's identity is a device certificate whose lifetime exceeds its `contact_max` plus a margin, renewed at contact; secrets for it are scoped to the device; revocation takes effect at its next contact and is shown as `pending-contact` until then | ADR 0007 a4; ADR 0016 d4; ADR 0024 d6 |
| L35.6 | P5 | A 4 GB image went over L-band at satellite prices when two layers had changed. | **rule** (applied: ADR 0031 rules; ADR 0011 rules; ADR 0026 rules; ADR 0010 rules): transfers to and from edge computers use the link class their urgency allows (`--link=port\|leo\|any`), are priced per link class in `explain`, and move images by changed layers only | ADR 0011 d7 (L10.5); ADR 0026; ADR 0010 |
| L35.7 | P2 | `kill` to a process on a buoy out of contact returned 0, and the run finished and reported success. | **rule** (applied: ADR 0031 rules; ADR 0008 rules; ADR 0011 rules; ADR 0004 rules; command map): a signal to a process on a computer out of contact returns at once with `queued until contact`; `wait` and the receipt record when the signal was delivered and when the process exited, and which came first | ADR 0008; ADR 0011 a4; ADR 0004 |
| L35.8 | P3 | A replaced server left four processes pending for ever, and its disk held the only copy of 11 days of video. | **rule** (applied: ADR 0031 rules; ADR 0011 rules; ADR 0006 rules; ADR 0024 rules; command map): `replace <old> <new>` carries an edge computer's group, desired state and pending work to the new device, not its data; outputs left only on the old device are listed as at risk; processes on a device not heard from for `contact_max` become `lost` | ADR 0011 d2 (L10.8); ADR 0006; ADR 0024 d6 |
| L35.9 | P6 | A vessel showed "ready, 1 GPU free" from nine days ago, and new work was placed on it. | **amend** (applied: ADR 0011 a4 revised) ADR 0011 amendment 4: a computer that connects by contact is `out-of-contact` between contacts, a normal state with its last and expected next contact, not `disconnected`; listings always print the age of what they show; placement uses state older than `contact_stale` only for work that can wait for the next contact; the alert is raised when a contact is late by `contact_grace`, not when the device is merely between contacts; queued commands expire after `contact_max` | ADR 0011 a4; ADR 0025 d2 |
| L35.10 | P9 | Crew ran their own video tool on a vessel's GPU, and Convoy counted the GPU as free. | **rule** (applied: ADR 0031 rules; ADR 0024 rules; ADR 0025 rules): the edge driver reads the device's full list of workloads at each contact; those Convoy did not deploy are shown as `outside Convoy` with their resource use, count against capacity, and are never removed by Convoy | ADR 0024 d8; ADR 0025 d2 |
| L35.11 | P1 | Convoy's deployment replaced another team's deployment for the vessel group, and their collector disappeared from 38 ships. | **rule** (applied: ADR 0031 rules; ADR 0010 rules; ADR 0025 rules): on an edge computer run by a fleet manager, Convoy writes its intent as the fleet manager's deployment, merged with the components others declared, never replacing them; the fleet manager is the computer's operator (ADR 0025 decision 8) | ADR 0010 a1; ADR 0025 d8 |
| L35.12 | P10 | A master stopped inspection during a storm, and Convoy restarted it 40 minutes later. | **rule** (applied: ADR 0031 rules; ADR 0021 rules; ADR 0008 rules; ADR 0017 rules): a stop by a local authority on the device (a master's panel, a site's safety switch) shows the work as `stopped locally` with the reason the device reports, and Convoy does not restart it until the device reports the stop lifted | ADR 0021 d10; ADR 0008; ADR 0017 |
| L35.13 | P12 | A 24-hour deadline was inherited by onboard stages on ships that call home every few days. | **rule** (applied: ADR 0031 rules; ADR 0004 rules; ADR 0017 rules): children on edge computers count their deadlines in contacts (`--contact-deadline=<n>` or a duration from first contact), bounded by the tree's deadline; the central stages proceed on what is delivered by the floor | ADR 0004 a3; ADR 0017 d7 |
| L35.14 | none | Buoys started a 30-minute stage at 38 % battery and stopped at 20 %, fourteen times. | **rule** (applied: ADR 0031 rules; ADR 0025 rules): a computer with a power budget reports battery, threshold and forecast in `bounds`; a stage declares `--energy=<Wh>`; the device starts it only when the budget allows, and `ps` shows `waiting-power` | ADR 0025 d2, d4 |
| L35.15 | none | Buoy receipts arrived with times up to 14 minutes off, some in the future. | **rule** (applied: ADR 0031 rules; ADR 0009 rules): a receipt from an edge computer carries `clock=synced\|device`, the device's offset measured at the next contact, and times corrected by it, with the uncorrected value kept | ADR 0009; ADR 0009 a1 |
| L35.16 | none | An image with no arm64 variant was downloaded by 228 camera boxes that could not run it. | **rule** (applied: ADR 0031 rules; ADR 0025 rules; ADR 0026 rules; ADR 0022 rules): the plan checks an image's platforms against each device's architecture (the `image=` field of ADR 0025 decision 2) before any transfer, and refuses with exit 3, listing the devices it cannot run on | ADR 0025 d2; ADR 0026; ADR 0022 |
| L35.17 | none | A signing key was rotated, and ships at sea with the old trust root refused every new image. | **rule** (applied: ADR 0031 rules; ADR 0026 rules; ADR 0007 rules): a new trust root is delivered to edge computers before any image signed only with the new key is released to them; `explain` lists devices that do not yet hold it | ADR 0026; ADR 0007 |
| L35.18 | none | A vessel never called home again. | **accepted**: a computer out of contact can be told nothing; work, signals and revocations take effect at its next contact, and Convoy promises nothing about a device that does not return beyond marking its work `lost` after `contact_max` and listing what was only on it | ADR 0011 a4 |
| L35.19 | none | Footage recorded in one state's waters, on a ship flagged in another, carries only `residency=eu`. | **accepted**: Convoy enforces the residency the data's owner declares and records the device's location with each output; it does not decide which jurisdiction's law applies at sea | ADR 0016 d11; L26.4 |
