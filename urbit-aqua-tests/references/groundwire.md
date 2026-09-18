# Groundwire-specific Aqua tests

Read this only for Groundwire Urbit or its identity-provider fixtures. Ordinary
Urbit tasks should use `ph-io` and the Azimuth reference without these imports.
Names and details below describe the inspected Groundwire tree; inspect the
target revision before applying them.

## Select a coherent source and runtime

Groundwire changes more than an application desk. Its Aqua host agent, Jael
interfaces, key fixtures, boot code, and networking helper differ from ordinary
Urbit. Use the matching Groundwire Arvo sources and a compatible Vere runtime
for the host and guest pill. Do not mix ordinary `/ted/aqua/ames` with Groundwire
vane or surface definitions because the filenames happen to match.

Inspect:

- `/app/aqua`, `/sur/aquarium`, and `/lib/aqua-azimuth`;
- `/lib/ph/io`, `/lib/ph/util`, `/lib/ph/gw/io`, and `/lib/ph/gw/util`;
- `/ted/aqua/ames`, `/lib/aqua-vane-thread`, and `/sys/vane/ames`; and
- examples under `/ted/ph/gw`.

The Groundwire Ames helper tracks fiefs, sponsor chains from `%saxo`, lanes
from `%nail`, Mesa interests, and peer-knownness responses. Its Aqua boot path
loads the chosen `%ames`/`%mesa` core and initializes Khan. Use these supplied
helpers rather than creating application-specific variants or intercepting
network effects just to observe readiness. An independent Aqua watch can
observe effects without preventing the standard helper from receiving them.

Stage host dependencies from the configured Groundwire checkout through the
project's build workflow; do not hardcode a particular user's checkout path.
The guest pill still needs the application's own desk, bill, marks, and test
observers. Prefer the project's existing pill-loading command. The example
`/ted/ph/gw/load-pill` also modifies kiln to disable OTAs, so inspect that
behavior before choosing it over ordinary pill construction.

## Groundwire identities and helper API

The provider library is imported and instantiated as:

```hoon
/+  gw-io=ph-gw-io
::  inside a strand's setup:
=/  io  ~(. gw-io %.y %my-test)
```

The door arguments select logging and its tag. Its fixture directory is:

```hoon
+$  onchain
  (list [peer=@p =life =rift spon=(unit @p) fef=?(~ %turf %is %if)])
```

In actual Hoon declarations, use the checked-in `onchain:io` mold rather than
retyping it. Each row records a peer's key life, rift, optional sponsor, and
fief fixture choice. The label `onchain` does not imply that Aqua is contacting
a real chain: the fixture synthesizes Jael updates.

For a Groundwire-only comet topology, start helpers once with
`start-simple:io`, then boot each selected fixture comet with:

```hoon
(start-gw-comet:io who %mesa chain)
```

Select `%ames` or `%mesa` deliberately for the behavior under test. The helper
boots a non-fake comet, stages a tiny `%gw` source agent and the
`%groundwire-udiffs` mark into guest `%base`, starts `%gw`, and injects the
directory. This existing identity-fixture staging is an exception to the
general no-runtime-copying guidance; do not extend it to copying the
application-under-test into `%base`.

Use the comet identities present in `ph-gw-util` and `aqua-azimuth`'s key
fixtures. They include distinct crypto suites. A random `@p` is not a valid
replacement for a mined comet with matching feed/key material. `%if` and
`%turf` depend on fixture lookup tables; inspect `fief-udiff` before assuming
an arbitrary peer resolves. In the inspected version `%is` emits no fief,
not a complete implementation of a different fief mode.

Do not `spawn` Groundwire comets as Azimuth identities. For mixed topologies,
use `start-azimuth:io`, provision the Azimuth peers with `spawn:io`, and boot
them with `init-ship-core:io`; the Groundwire comets retain their own boot path.
See `/ted/ph/gw/az-star-hi` and `/ted/ph/gw/az-comet-hi`.

## Jael updates, key changes, and continuity

Use existing provider helpers instead of constructing large typed events:

- `poke-all-udiffs` supplies key, fief, rift, and sponsorship state for one peer;
- `poke-keys-udiff` distributes a new key life to a receiver;
- `poke-udiffs` supplies a selected update list;
- `poke-rekey` updates the guest's private feed through Hood; and
- `gw-breach` reboots with the new life/rift, reinstalls the provider fixture,
  distributes updates, and returns an updated `onchain` directory.

Keep local private-feed changes and peer public-key updates distinct. A
successful Hood rekey does not mean every peer's Jael source has the new key.
Likewise, do not reuse the old directory after `gw-breach` returns a replacement.
Inspect helper completion signals: current udiff helpers wait for Dojo output,
and `gw-breach` includes fixed sleeps. These are existing fixture behaviors,
not a pattern to copy into application polling or a claim of semantic readiness.
Validate the actual key/continuity/network behavior after provisioning.

## Sponsorship and routes

A fief-less sponsee may need to contact its sponsor first so the simulated
network can discover its route. Establish that directed sponsor path before
traffic to or from other peers. Use `send-hi:io` only on required routes; its
success establishes a request/reply round trip, not application readiness.
See `/ted/ph/gw/hi-sponsor` and the sponsor key-cycle/breach examples.

Do not replace necessary sponsor contact with a full all-to-all handshake
matrix. Conversely, do not omit it merely because the pill booted successfully.

## Restore readiness and source registration

The modified networking code emits ship-tagged `%saxo` while processing Ames
`%born`. For a revision that does so for every restored guest, a parent watching
`%aqua`'s `/effect` can wait for one from each expected ship without changing
`aqua-vane-thread` or `/ted/aqua/ames`. This is only an Ames restore boundary:
also verify the relevant application endpoint or semantic agent readiness.
Do not assume `%saxo` is a universal ordinary-Urbit readiness signal.

In the Chorus fixture work, Jael's live source registration/directory required
explicit reconstruction after restoration. Inspect the current restore path
and verify the required entries rather than always replaying everything. When
reconstruction is necessary:

1. Register the fixture `%gw` source using the current Jael `%listen` task.
2. Rebuild any ship-to-source index needed for directory queries.
3. Inject only the required peers' key/fief/sponsorship/rift updates.
4. Validate the expected source/directory state before testing the application.

Default-source registration may watch `%gw` at root; explicit per-ship source
indexing may additionally cause ship-qualified watches. Match the fixture's
`on-watch` contract to the actual Jael behavior. Do not confuse those provider
watch paths with the Aqua test operation wire, silently subscribe on arbitrary
paths, or modify the production provider to accommodate a mistaken test.
In particular, a test observing a ship must not turn the provider's default
root watch into `/<ship>` merely for correlation: use the test's Aqua wire.
Ship-qualified provider watches are appropriate only where Jael's actual
source-indexing contract requires them. See the shared
[paths and wires distinction](patterns.md#paths-and-wires).

Account for the standard vane helper's deferred outgoing effects between
snapshot epochs as described in `patterns.md`. Obsolete Ames bones and a
stalled `send-hi` immediately after restore can indicate lifecycle/source setup
errors rather than a broken application protocol.
