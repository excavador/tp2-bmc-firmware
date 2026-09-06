# LACP on this branch is KNOWN BROKEN — do not flash it

This branch carries upstream PR #254 (802.3ad link aggregation). It is
parked here so the work is *reviewable and buildable*, not because it is
ready. Upstream CI has never built it: the PR sits at `action_required`,
GitHub's first-time-contributor gate, and nobody approved it.

## What happens if you flash it

Tested by `hungerf3` (2026-01-18):

> LACP negotiation was successful, and when testing from the BMC, the other
> modes seemed to work. **Only the BMC network worked; the nodes couldn't
> talk to the outside world.** I saw the links come up, but they were unable
> to reach the DHCP server.

Diagnosed by `j0ju` (2026-01-26):

> This patch assumes that in kernel DSA is working with VLAN and
> link-aggregation support from the switch driver. **In the current state it
> does not work.** It just isolates ge0/ge1 from the node ports. So you have
> no connectivity at all.

On a 4-node board that is a total outage of every node.

## Two further objections, both unresolved

1. **It hardwires network configuration.** Per j0ju, even with working DSA
   the implementation does not allow VLANs on the bond, so it removes
   configuration a user can do manually today. A net loss of flexibility.
2. **It belongs in three PRs, not one** — bmcd, BMC-UI and this repo.

## What LACP would actually buy this estate

Less than it sounds, and nothing today:

- The Turing Pi 2 has **two** 1GbE external ports (ge0/ge1) into the
  on-board RTL8370, which also serves the four node slots. Bonding gives
  **2 Gbit/s aggregate for the whole board** — not per node.
- **No node can exceed 1 Gbit/s** regardless: each RK1 has a single 1GbE
  link to the internal switch.
- **No single flow exceeds 1 Gbit/s.** LACP hashes per flow, so one TCP
  stream picks one link. Two concurrent flows to different peers are needed
  before any gain appears.
- **Our switch cannot do LACP.** The TL-SG108E offers *Static LAG* only
  (2 groups, 4 ports each) — no 802.3ad. Read off the device, not the
  datasheet.
- Most hive traffic is pod-to-pod between nodes, which stays inside the
  on-board switch and never touches the uplink at all.

**Where it becomes real: board B.** With two boards, cross-board traffic
must traverse the uplinks, and 2 Gbit/s aggregate stops being theoretical.
That is the case worth fixing this for.

## Doing it properly

1. Establish whether the kernel's RTL8370 DSA driver supports LAG offload
   at all. If it does not, this is a kernel problem, not a config one.
2. Keep VLAN-on-bond working — see `topic/vlan`, which is the cleaner patch
   and testable end to end, because the TL-SG108E *does* do 802.1Q.
3. Test on board B's BMC first. Never on the only working one.
