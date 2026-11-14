# ADR 0031: An edge computer is reached by contact, and known by what it delivers

**Status:** Draft · 2026-11-14
**Supersedes / relates to:** open finding L35.1 (computers with no API that pull work at contact have no kind, contact model or delivery states); ADR 0025 decisions 2, 3, 4, 5 and 8 and its open item on computers at the edge with no provider API (the computer contract, the kinds, placement, no federated computer, external operators); ADR 0029 decisions 6 and 7 and its open item on edge computers holding the only copy (residency of staged data, a stage succeeds when its outputs arrive); ADR 0011 decisions 2 and 7 and amendments 1, 3 and 4 (`by-id`, `/sys`, drivers from packages, `residency`, `disconnected`); ADR 0009 amendment 1 (receipts spooled where work completes); ADR 0004 amendments 2 and 3 (the degraded channel, the inherited contract); ADR 0007 amendment 4 (site-local secret sources); ADR 0008 (signals); ADR 0016 decisions 4 and 11 (the exchange per open, residency); ADR 0021 decision 10 (people win); ADR 0024 decisions 6 and 8 (removal, devices not installed by a module); ADR 0026 (digests, layers, verification); US-10 and accepted limitation L10.8

## Context

Every computer in ADR 0025 accepts a submission: an API server, `slurmrestd`, a function's invoke call, a batch queue, an instance group. Some computers accept nothing. A ship's onboard server behind a satellite link, a camera box on a mast, a solar buoy with a few minutes of Iridium twice a day: each runs an agent that connects out, pulls what it should be running, runs it, and reports when it next connects. Between contacts it cannot be reached at all, and that is its normal state, not a failure. Its results often exist only on its own disk until the next port or the next good link.

US-10 covered edge sites that are single-node Kubernetes clusters with a link that drops; ADR 0011 amendment 4 made `disconnected` a state, and ADR 0009 amendment 1 spooled receipts at the site. US-35 took the rest: a release to 38 vessels "done" at 31, a 4 GB image over an L-band link when two layers had changed and with no arm64 variant for 228 camera boxes, another team's fleet deployment replaced, a master's stop overridden, crew workloads unseen, placement on nine-day-old state, `kill` to a buoy out of contact returning 0, crew CCTV uploaded to the nearest region, the only copy of three weeks of video lost with a disk at sea, a replaced server's work pending for ever, a certificate that expired on a long voyage, a trust root rotated under ships at sea, drifting clocks, buoys starting work below their power threshold, and a 24-hour deadline inherited by ships that call home every few days.

Facts that constrain the design:

- Edge devices are reached by pull: they connect out, through NAT, satellite or cellular, when they can and when their power allows. Fleet managers (AWS IoT Greengrass, Azure IoT Edge, balena and similar) already work this way, with deployments declared per group of devices and pulled at contact.
- A fleet is many computers with different states at any moment; its aggregate state is a count, never a single answer.
- Links differ by orders of magnitude in price and bandwidth, and the cheap one is not always there.
- A device's identity, clock and trust root are only as current as its last contact.
- Physical authority on board (a ship's master, a site's safety officer) outranks a remote plan.

## Decision

1. **`edge` is a computer kind for devices that accept no submission.** ADR 0025 decision 3's kinds gain `edge`: a device that pulls its desired state when it connects. Each device is one computer under `/dev/compute/by-id/edge-<manager>-<device>`; a fleet is a view (`by-label/fleet=vessels`), never one computer (ADR 0025 decision 5). The driver works through the device's fleet manager, which is the computer's operator (ADR 0025 decision 8), or through Convoy's own edge agent where there is none (decision 10). It fills every contract field: `stage` as a desired state per device, `status` with the states of decision 3, `signals=queued`, `image=` with the device's architectures, `bounds` with memory, accelerators and power, `identity=device`, `price` per link class, `queue` as the expected next contact, and `operator=` naming the fleet manager.

2. **Contact is in the contract, and out of contact is a normal state.** ADR 0025 decision 2's contract gains a field, `contact`: the last contact and the link it used, the expected next contact from the device's schedule or history, the link classes the device can use with their price and bandwidth, and the platform default `contact_max` for its fleet. `/sys/compute/<id>/contact` holds it:

   ```
   /sys/compute/edge-greengrass-vessel-17/contact
   last     2026-11-12T03:10Z  via leo
   next     about 2026-11-13T03:00Z  (daily, within 4 h, from 30 days of history)
   links    leo        metered per GB   about 20 Mbit/s   most days
            lband      metered per MB   about 64 kbit/s   always
            port-wifi  free             in port
   ```

   Between contacts the computer is `out-of-contact`, with its last and expected next contact (ADR 0011 amendment 4 as revised by L35.9). Everything shown about it carries its age; placement uses state older than `contact_stale` only for work that can wait for the next contact; an alert is raised only when a contact is late by `contact_grace`; queued commands expire after `contact_max`.

3. **Work is a desired state the device pulls, and it has states for the time in between.** Convoy writes a device's work as the fleet manager's deployment for that device or its group, merged with the components others declared and never replacing them. Each device's part of a process moves through `pending-contact`, `running`, `done-local` and `delivered`; a device whose contact is late by `contact_grace` is `silent`, one not heard from for `contact_max` is `lost`, and one stopped by a local authority is `stopped locally` (decision 6). A fan-out over a fleet view is one process with a floor and a deadline: `ps -T` shows each device's state, and at the deadline it exits 0 at the floor (degraded below 100 %, ADR 0004 amendment 2) or 1 below it, listing the devices not heard from. A child on an edge computer counts its deadline in contacts (`--contact-deadline=<n>` or a duration from first contact), bounded by the tree's deadline, and the central stages proceed on what is delivered by the floor.

4. **The only copy is a stated condition, and the device keeps it until it is delivered.** A stage on an edge computer is `done-local` until its outputs are delivered and verified at their destination (ADR 0029 decision 7), then `delivered`. The device never deletes outputs before delivery. `ls` of an output not yet delivered says `only copy on <device>`, and `explain` lists the bytes at risk per device. Outputs are uploaded only to stores within their residency, whatever is nearest (ADR 0029 decision 6), and each carries the device's reported location when it was written. `replace <old> <new>` carries a device's group, desired state and pending work to its replacement, not its data; outputs left only on the old device are listed as at risk.

5. **A transfer chooses its link and moves only what changed.** A transfer to or from an edge computer uses the link class its urgency allows (`--link=port|leo|any`, by the names the device reports), and `explain` prices it per link class. Images move by changed layers only, by digest (ADR 0026). Before any transfer, the plan checks the image's platforms against each device's architecture and refuses with exit 3, listing the devices it cannot run on.

6. **Signals and stops are messages that arrive at contact, and the people on board win.** A signal to a process on a computer out of contact returns at once with `queued until contact`; `wait` and the receipt record when the signal was delivered and when the process exited, and which came first. A stop by a local authority on the device (a master's panel, a safety switch) is reported at contact as `stopped locally` with its reason, and Convoy does not restart the work until the device reports the stop lifted (ADR 0021 decision 10).

7. **Identity is the device's, and it lasts the voyage.** An edge computer's identity is a device certificate whose lifetime exceeds its `contact_max` plus a margin, renewed at contact. Secrets for it are scoped to the device and come from a site-local source (ADR 0007 amendment 4). Revocation takes effect at its next contact and is shown as `pending-contact` until then; `rm` of an edge computer revokes it and has Convoy's own data on it wiped at that contact, never data whose only copy is there. Images are verified on the device with its cached trust root, and a new trust root is delivered before any image signed only with the new key is released to it; `explain` lists devices that do not hold it yet.

8. **What a device runs is read at contact, including what Convoy did not put there.** At each contact the driver reads the device's full list of workloads and their resource use. Workloads Convoy did not deploy (another team's, through the fleet manager's console, or the crew's) are shown as `outside Convoy`, count against capacity, and are never removed by Convoy.

9. **Power and clocks belong to the device, and receipts say how far to trust them.** A computer with a power budget reports battery, threshold and forecast in `bounds`; a stage declares `--energy=<Wh>`, the device starts it only when the budget allows, and `ps` shows `waiting-power`. Receipts are spooled on the device (ADR 0009 amendment 1) and carry `clock=synced|device` and the device's offset measured at the next contact; times are corrected by it, and the uncorrected values are kept.

10. **Where there is no fleet manager, Convoy's own agent does the same.** An edge agent package (a driver under ADR 0011 amendment 1) runs on devices with no fleet manager: it connects out to the controller in the device's residency, pulls the desired state, runs it with the device's container runtime, spools receipts and outputs, reports its workloads, power and clock at contact, and holds the device certificate. It is installed by a module (ADR 0024) and is the computer's operator in place of a fleet manager.

## Declined

| Proposal | Why not | What replaces it |
|---|---|---|
| an API on every device, pushed to | devices sit behind NAT, satellites and power budgets, and an open port on each is an attack surface | pull at contact (decisions 1, 3) |
| a fleet as one computer | its devices are in different states at every moment; ADR 0025 decision 5 | each device a computer, the fleet a view (decision 1) |
| out of contact as `disconnected`, alerted | it is the normal state; every buoy would alert twice a day | the contact model and late-contact alerts (decision 2) |
| deleting outputs on the device after the stage reports success | the device may hold the only copy | `done-local` until `delivered` (decision 4) |
| short-lived device credentials | they expire at sea | lifetimes beyond `contact_max`, renewal at contact (decision 7) |
| synchronised clocks as a precondition | not every device can reach a time source | the offset in the receipt (decision 9) |

## Consequences

- L35.1 is drafted here, and the open items on computers at the edge with no provider API (ADR 0025) and on edge computers holding the only copy (ADR 0029) are settled by it. US-35's other findings land on this record when US-35 is applied: L35.2, L35.11 and L35.13 (fan-outs, merged deployments, deadlines in contacts) in decision 3, L35.3, L35.4 and L35.8 (the only copy, residency and location, replacement) in decision 4, L35.6 and L35.16 (links, layers, architectures) in decision 5, L35.7 and L35.12 (queued signals, local stops) in decision 6, L35.5 and L35.17 (device identity, trust roots) in decision 7, L35.9 (an amendment to ADR 0011 amendment 4 that decision 2 assumes) in decision 2, L35.10 (workloads outside Convoy) in decision 8, L35.14 and L35.15 (power, clocks) in decision 9; L35.18 and L35.19 are accepted.
- ADR 0025 decision 2 gains the `contact` field and decision 3 the `edge` kind. ADR 0011 amendment 4's `disconnected` stays for devices that are normally connected, and `out-of-contact` serves those that are not. ADR 0029 decision 7's rule that a stage succeeds when its outputs arrive gains the `done-local` state. US-10's k3s sites remain `k8s` computers with links.
- New paths: `/sys/compute/<id>/contact`, `/sys/compute/<id>/power`, `/sys/compute/<id>/location`. New platform defaults: `contact_grace`, `contact_stale`, `contact_max`. New `caps`: `edge:<manager>`, `link:<class>`, `power:<source>`, `arch:<platform>`.
- ROADMAP: placement with contact age and power, link selection and the architecture check are M3; `done-local` and `delivered`, and images by changed layers, are M4; device certificates, revocation at contact and trust roots are M7; the contact model, late-contact alerts, `replace` and workloads outside Convoy are M8; fleet manager drivers and the edge agent are M9; spools, clocks and power on the device are M10.
- Open, to be settled against this record: devices that relay for each other (a ship's server for its camera boxes, a mesh of buoys); satellites as computers, with contact windows predicted by their orbits; operating-system and firmware updates, which fleet managers do and Convoy does not; a story that probes this record.
