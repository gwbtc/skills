---
name: urbit-aqua-tests
description: Write, review, and debug Aqua integration-test threads for ordinary Urbit or Groundwire Urbit, using ph-io and Spider strands for multi-ship userspace, PKI, key, and continuity scenarios. Use for Aqua test design or implementation, not ordinary Hoon unit tests.
---

# Urbit Aqua Tests

Build deterministic integration tests around observable protocol completion,
using the standard Aqua helpers and the smallest network topology that proves
the behavior.

## Before editing

Inspect the target repository's current:

- `/lib/ph/io.hoon` API;
- `/app/aqua.hoon` and `/lib/aqua-vane-thread.hoon`;
- `/ted/aqua/{ames,behn,dill,eyre}.hoon` helper threads;
- nearby Aqua threads and test-observer agents;
- the project's pill composition and build workflow; and
- desk bills and marks required inside guest ships.

Use the checked-in versions as authoritative. Aqua and `ph-io` interfaces can
differ between Arvo revisions.

### Ordinary Urbit or Groundwire

Establish which Arvo/Vere variant the target ship and guest pill actually use;
do not infer it from a ship name or assume the host's dependency checkout is
the right one. Use ordinary `ph-io` and the Azimuth reference for regular
Urbit. Only when working on Groundwire Urbit or its PKI fixtures, read
[references/groundwire.md](references/groundwire.md). It covers `ph-gw-io`,
custom Jael sources, Groundwire comet identities, and modified Aqua networking.
Regular Urbit tasks need not load that reference or import Groundwire libraries.

## Preferred structure

Use `spider`, `strand`, and `*ph-io`. Choose the ship model before writing the
setup:

- **Fake ships:** for userspace-agent behavior that needs network delivery but
  does not test real PKI decisions. Use `start-simple` and `(init-ship who &)`.
- **Simulated Azimuth identities:** for Azimuth-derived PKI, sponsorship, keys,
  or continuity behavior. Use `start-azimuth`, `spawn` each Azimuth identity,
  then `(init-ship who |)`. See the Azimuth reference for moons and ordinary
  comets, which do not all follow the planet provisioning sequence.
- **Other non-fake PKI fixtures:** use the identity provider under test, not
  Azimuth by default. Groundwire-only comet fixtures use `start-simple`,
  `start-gw-comet`, and Jael udiffs; they do not use Azimuth `spawn`. Read the
  Groundwire reference only when that variant/provider is relevant.

`start-simple` selects helper startup, not fake identity state: the boot helper
and its `fake` argument determine that. Non-fake does not mean Azimuth-backed.
A mixed-provider topology may require both providers' provisioning paths.

Do not use fake ships merely because setup is shorter when fake identity state
would bypass the behavior under test. Conversely, do not pay for Azimuth
simulation when an isolated userspace protocol test does not observe PKI.

For an ordinary Ames/Gall test after selecting the ship model:

1. start the appropriate Aqua mode;
2. provision and initialize each ship as required by that mode;
3. start or configure only the required agents;
4. connect only peers that must communicate, using `send-hi`;
5. register the expected completion before triggering the operation;
6. trigger it with `poke-app` or a typed Dojo command;
7. wait for a semantic observer signal or exact output; and
8. finish with `end`, respecting the current helper/Spider lifecycle.

Check whether `stop-threads` actually stops helpers. In revisions where it is
a no-op, start one helper set per parent suite runner, not per test case.
`end` closes the caller's effect watch; it does not kill the helpers or drain
pending effects. Spider cleans up those children when their owning parent
finishes or fails. See
[references/patterns.md](references/patterns.md#helper-and-snapshot-lifecycle).

Keep the application-under-test and its test observers in desks included in the
Aqua pill. Do not merge application source into `%base`, dynamically copy it
into guests, or create application-specific vane-helper variants when the
helpers supplied by the selected Arvo tree provide the required behavior.
Existing identity-provider fixture boot helpers may stage their own support
files; this is not a reason to copy the application-under-test at runtime.

Avoid hand-constructed `%aqua-events`, especially for Gall pokes. Use `ph-io`
operations such as `init-ship`, `send-hi`, `poke-app`, `dojo`, and
`wait-for-output`.

This is not only a convenience rule. Injecting a typed poke as a raw Aqua event
can embed and copy the host's `vase` type noun into the event delivered to a
virtual ship. In Hoon, that inferred type structure can be vastly larger than
the value and can make an otherwise small poke consume extreme CPU and memory.
`poke-app` instead writes the value through the virtual ship's Dojo, so parsing,
mark conversion, and type checking happen inside that ship without transporting
the host's type structure.

Do not inject application pokes as raw Gall tasks carrying a host `vase`.
Prefer the existing Dojo/poke helpers. A small typed text adapter is acceptable
when a helper loses the payload's auras; it still sends Dojo text, not a raw
typed Gall poke. Reserve hand-constructed events for low-level operations not
represented by the selected `ph-io`, keeping their payloads small.

## Synchronization

After snapshot restoration, require a fresh, ship-specific semantic readiness
signal for every guest used by the test before starting tested operations.
Restore acknowledgement means the fleet was installed, not that restored
vanes, PKI sources, or agents are usable. Match readiness to the dependencies
the scenario needs; Dojo arithmetic proves only that Dill/Dojo can evaluate
input. See the
[restore-readiness pattern](references/patterns.md#restore-readiness-pattern).

Synchronize on causally meaningful output, not elapsed time. A good pattern is
a small guest observer agent that:

- accepts an expectation before the tested operation starts;
- receives the tested agent's typed callback;
- validates the result on the guest; and
- prints one unique completion line that `wait-for-output` consumes.

This avoids host-side scry polling, repeated timers, and large nouns crossing
the Aqua boundary. If `ph-io`'s `wait-for-fact` directly matches the interface,
it is also appropriate.

A callback observer is not mandatory. Another valid non-invasive pattern is
to open a fact watch before the operation, observe a causal application fact,
then run a one-shot generator inside the guest to validate typed final state
and emit only a small unique success result. The fact must establish the state
boundary the probe needs; this is event-driven validation, not remote-scry
polling. See the
[fact-watch and guest-probe pattern](references/patterns.md#fact-watch-and-one-shot-guest-probe).
Put every potentially unbounded wait under a whole-test timeout.

Keep the test's Aqua event/effect wire, the Gall `%watch` path delivered to
`on-watch`, and any application callback/reply path distinct. Put test
operation IDs on the correlation wire where the API permits, not in an
application subscription path unless that application's contract requires it.
In particular, identifying a guest ship does not require subscribing at
`/<ship>`. See
[paths and wires](references/patterns.md#paths-and-wires).

Do not poll Gall scries through Aqua on periodic sleeps. Repeated sleeps create
Behn events and repeated virtual-state inspection can consume extreme CPU and
memory. `<<behn>>` lasting unexpectedly long is a prompt to look for polling,
timer storms, oversized effects, or very long protocol timeouts—not to add more
sleeps.

Configure application request timeouts to a short test-appropriate duration,
but let the application own those timers. Delays are not normal readiness or
completion mechanisms. Exception: after pausing the old fleet, a short,
bounded, documented teardown grace may allow known deferred helper cards to
drain before replacing its snapshot epoch. This is lifecycle isolation, not
polling, readiness, or an assertion that an operation succeeded. Prefer an
observable drain if available; do not universalize a measured duration. See
[teardown drain](references/patterns.md#teardown-drain-exception).

## Typed interaction

Use the application's real command mark and typed command noun. Prefer:

```hoon
(poke-app who %my-agent %my-command command)
```

or, when a Dojo-rendered typed command is specifically useful:

```hoon
(dojo who ":my-agent &my-command {<command>}")
```

Use regular `poke-app` whenever it renders the payload sufficiently; do not
introduce custom wrappers merely because its argument is `*`. Inspect its
signature and renderer when diagnosing malformed Dojo input or mark-cast
failures. Versions accepting `data=*` retain the noun's value but erase its
formatting type before `{<data>}` renders the command. Terms, ships, dates, and
other aura-sensitive fields may therefore be printed in a form that the guest
cannot reconstruct as the intended command. Casting the value at the call site
does not preserve that type through an untyped gate.

Only if regular `poke-app` is insufficient, write a small command-specific
adapter accepting the application's actual command mold. Construct the Dojo
input while the payload is still typed, using `{<command>}` or explicit `scow`
formatting as needed, then send that text through `dojo`. Keep mark conversion
and validation in the guest. See the typed-adapter example in
[references/patterns.md](references/patterns.md#typed-rendering-fallback).

Do not replace these with a raw event containing `!>(command)` or another host
`vase`: that transports the inferred type as well as the value. Ensure every
custom mark and its dependencies are present in the guest desk. For Gall `%gx`
scries, append the expected return mark to the path.

`wait-for-fact` receives and returns a bare noun, not a vase. Check the fact
mark and structurally refine the payload with shape/tag tests before slot or
field access; a matching mark alone does not establish its Hoon type. `!<`
requires a real vase and cannot decode this bare noun. `;;` performs runtime
mold validation and is substantially more expensive, not an equivalent
fallback. Prefer targeted structural checks in hot observers or typed
validation inside the guest. See
[raw-fact handling](references/patterns.md#raw-fact-handling).

## Validation and debugging

For aggregate runners, read the
[suite-level correctness checklist](references/patterns.md#suite-level-correctness).
Overall success must account for every selected test, including build/setup
failures and timeouts, and cases must remain isolated after failures.

Build the thread and every guest app, mark, and observer before running it.
Build the complete composed desk, not merely the local source fragment. Ensure
the current solid Aqua pill contains the application and test desks.
Host helper imports resolve from the host thread's Clay beak, without `%base`
fallback; guest pill contents do not satisfy them. Validate both dependency
sets separately; see
[desk and pill composition](references/patterns.md#desk-and-pill-composition).

When a test stalls, locate the last causal boundary: helper startup, ship boot,
agent boot, neighborhood establishment, poke acceptance, peer request,
response, callback, observer validation, or output wait. Add temporary logs at
that boundary rather than broad timer polling. Treat mark cast failures,
unexpected pokes, and missing desk files as composition/type errors before
investigating networking.

Read [references/patterns.md](references/patterns.md) when implementing a new
thread or diagnosing a hang; it contains a skeleton, observer pattern, and
failure checklist. Read [references/azimuth-ships.md](references/azimuth-ships.md)
when ordinary Urbit's Azimuth provisioning, moons, comets, keys, or continuity
are in scope. Read the Groundwire reference only for Groundwire-specific work.
