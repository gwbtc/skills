# Aqua thread patterns

Use these patterns as shapes, not as substitutes for inspecting the repository's
current `ph-io` implementation and application molds.

## Minimal fake-ship skeleton

This shape is for userspace networking where real identity state is not part
of the assertion. For identity-dependent behavior, use the appropriate
provider's non-fake setup: [azimuth-ships.md](azimuth-ships.md) for ordinary
Urbit, or [groundwire.md](groundwire.md) for Groundwire fixtures. Run this shape
inside a bounded test runner and adapt its command rendering to the current API.

```hoon
/-  spider, *my-agent-sur
/+  *ph-io
=,  strand=strand:spider
^-  thread:spider
|=  argument=vase
|^
=/  m  (strand ,vase)
;<  ~  bind:m  start-simple
;<  ~  bind:m  (init-ship ~bud &)
;<  ~  bind:m  (init-ship ~wes &)
;<  ~  bind:m  (start-agent ~bud)
;<  ~  bind:m  (start-agent ~wes)
;<  ~  bind:m  (configure ~bud ~wes)
;<  ~  bind:m  (send-hi ~bud ~wes)
;<  ~  bind:m  (expect ~bud 0v1)
;<  ~  bind:m  (poke-app ~bud %my-agent %my-command [%begin 0v1 ~wes])
;<  ~  bind:m  (await ~bud 0v1)
;<  ~  bind:m  end
(pure:m !>(~))
::
++  start-agent
  |=  who=@p
  =/  m  (strand ,~)
  (dojo who "|start %my-agent %my-desk")
::
++  configure
  |=  [a=@p b=@p]
  =/  m  (strand ,~)
  ;<  ~  bind:m  (poke-app a %my-agent %my-command [%set-timeout ~s10])
  (poke-app b %my-agent %my-command [%set-timeout ~s10])
::
++  expect
  |=  [who=@p id=@uv]
  =/  m  (strand ,~)
  (poke-app who %my-test-observer %noun [%expect id])
::
++  await
  |=  [who=@p id=@uv]
  =/  m  (strand ,~)
  (wait-for-output who "my-test-observer /operation/{(scot %uv id)} complete")
--
```

If agents are installed from the guest desk's bill during boot, explicit
`|start` calls may be unnecessary. If startup must be synchronized, wait for an
output the selected startup command actually emits on that Arvo revision. Do
not invent a readiness scry loop.

## Observer design

The observer is intentionally small. Its state normally maps operation IDs to
expected summaries. It receives the application's real result mark, validates
the fields relevant to the scenario, removes the expectation, and prints a
stable completion line.

Keep assertions on the guest when the result is large or typed. Send only the
success/failure summary through Aqua output. Register the expectation and any
callback before starting the operation so a fast result cannot race ahead of
the test.

An agent that already supports `[recipient reply-path]` callbacks should be
tested through that API. Do not repeatedly scry its completed-operation map.

Do not change production agents merely to give tests acknowledgements. A
callback observer is preferred when the API naturally supplies the tested
result, but a fact watch plus guest probe is also valid. An expectation helper
must actually register a watch or expectation; a no-op gives false confidence
about races.

### Paths and wires

These are separate fields with different owners, even though each can be
represented as a Hoon path:

| Field | Purpose | Who defines it |
| --- | --- | --- |
| Aqua event/effect wire | Correlates the guest task and resulting effects with a test operation | Test runner, within the selected helper's wire conventions |
| Gall `%watch` path | Selects the agent's subscription interface; delivered to `on-watch` | Subscribed agent's API |
| Application callback/reply path | Routes an application result to the supplied recipient/interface | Application callback contract and recipient |

For example, a test may assign `/g/aqua/watch/my-test/17` as its operation wire
while asking a guest agent to watch `/` (or its documented `/updates` path).
The agent receives that subscription path, not the operation wire. The guest
ship is a separate event target; neither its identity nor `17` belongs in the
subscription path merely to make the test identifiable. These wire/path
examples are illustrative, not universal helper syntax.

If an application accepts `[recipient reply-path]`, pass the callback path its
recipient expects. Do not substitute the Aqua wire for that path. Application
APIs may legitimately use IDs or ship-qualified paths; preserve those contracts
rather than imposing test correlation conventions on them.

Inspect helper signatures and task builders to verify which argument fills
which field. Match facts using the intended guest and correlation wire, then
validate their mark/payload. Unexpected `on-watch` paths usually call for
fixing the watch construction, not broadening the production agent's accepted
paths. For payload handling, see the raw-fact guidance below.

### Raw-fact handling

In the inspected `ph-io`, `wait-for-fact` accepts a predicate
`$-([mark noun] ?)` and returns `noun`. The `%raw-fact` payload has no attached
Hoon type. The helper matches ship, wire, and mark before invoking the
predicate, but that mark match does not validate the payload's structure or
refine its inferred type. Other custom effect consumers must check those
envelope fields themselves.

In the predicate or consumer, check the expected mark, then use structural
tests such as `?=` and tag checks to refine the noun before any nested slot
access. Guard each required cell/atom boundary; matching only the outer tag
does not prove the remaining fields are safe. After structural refinement,
validate the operation-specific values. Auras are formatting types, not
runtime atom tags, so an atom shape check alone does not validate a ship,
date, signature, or other domain-specific value.

`!<` extracts a value from a vase using its attached type; it cannot decode a
bare fact noun. Wrapping the payload with `!>` merely attaches its current
inferred type (often `*`), not the application's intended type, and does not
solve validation. Do not reinterpret an arbitrary cell as a vase either.

`;;` applies runtime mold validation/coercion and is substantially more
expensive than vase extraction. It is not an interchangeable fallback for
`!<`. Use it only when that full runtime validation is deliberately needed,
not indiscriminately for every fact in a hot matching loop. For large or
complex application state, prefer the one-shot typed guest probe below and
keep the host's checks limited to the causal fact's required structure and
correlation values.

### Fact watch and one-shot guest probe

This pattern observes the production agent without adding callback hooks:

1. Open the agent's real Gall fact watch before triggering the operation.
   Confirm subscription acceptance and install a consumer that retains fast
   facts while other acknowledgements are awaited.
2. Trigger the operation once through `poke-app` or a typed Dojo adapter.
3. Match a causal application fact from the intended guest and watch wire,
   checking the mark, sender, and operation-specific payload as appropriate.
   Ignore unrelated facts and initial state-on-subscribe snapshots that do not
   establish completion of this operation.
4. Run a test-only generator once inside that guest. It scries the relevant
   typed application endpoint, validates the expected final state, and emits
   a small unique success result only after the assertions pass. Include the
   generator and its imports in the guest pill; report failed assertions as
   failures, not as success output or invitations to retry.
5. Consume the guest/operation-tagged result and close the watch when done.
   Bound the entire sequence, including fact and output waits, with a timeout.

Inspect the agent's fact-emission semantics: a fact must establish the state
boundary needed by the probe, not merely announce that a request was received.
If it precedes the relevant state transition, choose a later completion signal
instead. Use distinct payloads or IDs so an old matching fact cannot pass the
test; keep application subscription paths separate from test correlation wires.

The causal fact is the synchronization mechanism; the single guest-local scry
is an assertion after that boundary. There is no timer-driven retry and no
repeated remote inspection waiting for state to change. Large typed state
stays inside the guest rather than crossing Aqua as a host vase. If the fact
already carries everything needed for a trustworthy assertion, validate it
directly and omit the redundant probe.

## Desk and pill composition

Host compilation and guest execution have separate dependency requirements.

The host desk containing the test thread needs:

- the test thread's libraries and surfaces;
- the selected Arvo tree's `/ted/aqua/{ames,behn,dill,eyre}.hoon` helpers;
- `/lib/aqua-vane-thread.hoon`, which those helper threads import with
  `/+ aqua-vane-thread`; and
- all their transitive imports.

With the inspected `ph-io` startup path, helpers receive the host thread's
`byk` (Clay beak: ship, desk, and revision). Their files and imports resolve
from that beak, not from a virtual guest's filesystem. Clay does not fall back
to `%base` when an import is missing from the selected desk. Including a file
in the guest pill does not satisfy a host-thread import.

For example, running a test from `%chorus` requires the helpers and
`/lib/aqua-vane-thread` in its selected host revision, even if `%base` has them
and the guest pill includes `%base`. Stage the matching dependencies through
the build workflow before committing the host desk rather than maintaining
modified copies. Check this separately from guest pill composition.

The guest pill separately needs:

- the application desk;
- every Gall agent in its bill;
- command, message, and result marks;
- test observer apps and their surfaces; and
- the standard runtime support expected by Aqua.

Build dependencies with the project's existing workflow, then commit any
mounted desks before rebuilding the pill. Prefer the project's established pill
command. A test should not mount a desk and copy application files into `%base`
as part of normal setup.

A host Clay build failure for `/ted/aqua/*` or `/lib/aqua-vane-thread` calls
for fixing the composed host desk, not rebuilding only the guest pill.
Conversely, successful host compilation does not prove that guest apps, marks,
or observers are present; verify the pill contents independently.

## Helper and snapshot lifecycle

Inspect `start-test`, `end-test`, `stop-threads`, and Spider child cleanup in
the selected revision. In both ordinary and Groundwire revisions encountered
here, `stop-threads` is a no-op. `start-simple` launches four long-lived child
threads for Ames, Behn, Dill, and Eyre; each owns its own Aqua subscriptions.
`end` removes only the calling thread's `/effect` watch, leaving those helpers
and their subscriptions alive. This is revision-specific, not Groundwire-only.

Calling `start-simple` / `end` around every case within the same parent can
accumulate helper sets. Old and new helpers can process the same effects,
duplicating packet forwarding, timer handling, or output. A standalone thread
can hide the issue by returning immediately after `end`: Spider then removes
the parent's descendant threads and issues leaves for their subscriptions.
Completing a per-test child does not remove helpers owned by the suite parent.

For these revisions, start one helper set in the parent runner, restore and
pause guest fleets between cases, then call `end` once and finish the parent.
`start-azimuth` already calls `start-simple`; do not call both to start helpers.
Guest pause/restore does not stop host helpers, and `end` is neither a helper
shutdown acknowledgement nor proof that pending effects have drained.
Do not implement explicit stopping from assumed helper IDs: verify that the
startup API actually requests or returns the running Spider IDs and inspect
its parent/child ownership.

Snapshot creation can clear the live fleet; do not assume a second topology
can be built by extending the first snapped one. Use separate snapshots for
different minimal topologies and rebuild them after changing guest code.

### Teardown drain exception

Pause the old fleet before replacing it. In the inspected standard
`aqua-vane-thread`, nonempty cards from a guest effect are deliberately emitted
after `sleep ~s0` to avoid inverting subscription flow. Zero seconds still
yields to a later event; it does not mean those cards have already been sent.
Replacing the fleet in that interval can deliver old traffic into new Ames
flows and produce stale/obsolete bones.

Prefer an observable drain barrier where the implementation provides one. If
the race is demonstrated and no drain acknowledgement exists, a short, bounded
grace after pausing the old fleet and before restoring the next is legitimate.
Document the deferred work, the evidence requiring the grace, and the chosen
duration at the call site. The Chorus runner used `~s1`; that is a measured
fixture workaround, not a default for every Aqua suite or a guarantee that
arbitrary queues have drained. Revisit it if helper scheduling changes.

This delay isolates snapshot epochs. It must not replace a semantic readiness
signal, become a repeated sleep/scry polling loop, or serve as an assertion of
success or non-delivery. Preserve the test's actual result independently of
the teardown delay, and still perform fresh per-ship readiness checks after
restoring the next fleet. `end` is not a substitute for this drain.

### Restore-readiness pattern

In the inspected Aqua implementation, `%restore-snap` installs the saved fleet
and publishes `%restore` effects. The host helpers subsequently process these
effects and inject vane `%born` events. The restore acknowledgement does not
wait for that work or for agents and identity sources to become usable.

Use this sequence under a bounded restore/setup timeout:

1. Establish the readiness observer before requesting restore. Confirm the
   subscription is active and the continuation can retain signals arriving
   before the restore acknowledgement; opening a watch alone is not enough
   if the caller discards unmatched facts while waiting for another response.
2. Request restoration and collect the selected revision's fresh, ship-tagged
   restore/readiness signals. Track a pending set of expected ships: repeated
   signals from one ship must not count as readiness for another. Ensure old
   fleet effects cannot satisfy this barrier, using an epoch/operation ID when
   supported or verified old-fleet teardown and effect isolation otherwise.
3. For each guest the test uses, validate the actual dependencies needed by
   the scenario. A networking restore effect may establish only the Ames
   boundary, not Jael source state or Gall/application readiness. Reconstruct
   external fixture state only where the inspected restore path requires it.
4. Obtain a semantic application signal: an existing callback or readiness
   fact, or a one-shot test-only guest probe that validates the relevant typed
   endpoint and restored state and emits a unique ship/operation-tagged result.
   A failed probe is a setup failure, not a reason to start a polling loop.
5. Only after all required guests pass the barrier, establish any necessary
   directed network routes and begin the tested operation.

The readiness predicate should cover the scenario's dependencies, not claim
that every vane is globally ready. A Dojo prompt or `(add x y)` result proves
only that Dill/Dojo can evaluate input; a successful `|hi` establishes a
network round trip, not application state readiness. Neither substitutes for
the per-ship semantic checks. Do not replace them with a fixed sleep.

Use an independent Aqua effect watch or the existing runner's verified effect
consumer; do not intercept effects or copy a modified vane helper just to
observe readiness. There is no universal restore-ready effect across Arvo
variants. Groundwire's `%saxo` boundary and Jael source restoration are covered
in [groundwire.md](groundwire.md#restore-readiness-and-source-registration).

## Interaction choices

Use `poke-app`, when its renderer is suitable, for application commands. It writes a
Dojo poke using the selected mark, so it exercises mark conversion and Gall in
the same way as a user-facing command. The virtual ship reconstructs the type;
the host does not send a `vase` containing its potentially enormous inferred
type noun.

### Typed-rendering fallback

Some ordinary and Groundwire versions accept `data=*` and render `{<data>}`.
The noun's bits survive, but the renderer no longer knows its auras. For
example, a term or ship inside a command can appear as an untyped atom instead
of the intended Hoon literal. A top-level atom-rendering workaround in one
revision does not necessarily restore typed rendering of nested fields.

First use the standard `poke-app`; inspect the generated Dojo input if the
guest cannot cast it with the real command mark. Check the noun and mark
dependencies too: not every cast failure is a rendering problem. Only when
the standard helper is insufficient, add a small typed adapter like this
illustrative arm, substituting the application's real mold, agent, and mark:

```hoon
++  poke-my-command
  |=  [who=@p command=my-command:my-agent-sur]
  =/  m  (strand ,~)
  ^-  form:m
  (dojo who ":my-agent &my-command {<command>}")
```

Here rendering occurs inside the gate with the actual command type, before
the text crosses the Aqua boundary. Use explicit `scow` formatting where the
typed pretty-printer still does not produce suitable input. Merely changing a
wrapper's parameter to `*`, or passing the typed value back into `poke-app`,
reintroduces the erasure. Reuse an existing suitable typed helper before
writing another one; do not replace functioning `poke-app` calls wholesale.

This remains a normal Dojo poke using the real command mark. It is not a raw
Gall event carrying `!>(command)`, and does not itself prove poke completion;
continue to wait for the test's semantic acknowledgement or observer result.

### Raw events and other helpers

Do not implement a poke with `send-events` and `!>(command)`. Raw Aqua event
injection may copy that full type structure through the host and virtual-ship
event machinery, turning a tiny command value into a very large event and
causing disproportionate CPU and memory use. This failure can look like slow
Ames or Behn processing even though the network protocol is not responsible.

Reserve hand-constructed `send-events` for low-level Aqua operations not
represented by `ph-io`. Keep payloads small and avoid embedded application
vases. An adapter using `ph-util`'s existing Dojo event builder still transports
text rather than a raw application poke; prefer `dojo` unless the adapter must
install its wait continuation atomically with the emitted event.

Use `dojo` for commands such as `|start`, `|rein`, and diagnostic expressions,
or when the exact textual interface is under test. Use `send-hi` rather than a
manual `|hi` plus a separate guessed delay.

Do not mechanically handshake every ordered pair. One successful `send-hi`
proves a request/reply round trip; a reverse `|hi` merely to prove that same
reachability is usually redundant. Explicitly test both directions only when
the scenario requires them. It does not acknowledge later application pokes.

Use `wait-for-output` for a unique line emitted after semantic validation. Use
`wait-for-fact` when the test naturally observes a subscription fact and its
wire/mark can be matched exactly. Avoid generic strings such as `"complete"`
that another ship or operation could emit.

## Failure checklist

### The thread stalls during startup

- Confirm the selected mode's `start-simple` or `start-azimuth` completed before
  ship initialization.
- For Azimuth identities, confirm `spawn` completed before non-fake boot.
  For Groundwire fixtures, check their own provider setup instead.
- Confirm the standard Aqua helper threads build from the host desk.
- Confirm the guest pill includes the intended desk and bill.
- Check whether the agent was already installed at boot.
- Verify that the waited-for startup text is actually emitted.

### `spider` reports a nonexistent helper or thread

- Use the standard helper names expected by the current `ph-io`.
- Do not stop helpers mid-test.
- Check that `/ted/aqua/<helper>.hoon` exists in the host desk.
- Check its libraries, especially `/lib/aqua-vane-thread`, in the same beak.
- Check for late cleanup signs versus a missing helper during active execution.
- Remove stale custom helper implementations before debugging the app.

### A poke cast fails

- Compare the command noun with the command surface mold.
- Confirm the mark's `++grab` converts the supplied source mark correctly.
- Ensure the mark resolves from the guest desk.
- Prefer a typed Hoon value over manually constructing the raw noun shape.
- Check whether `poke-app` erased auras while formatting `data=*`.
- Keep the poke on the `poke-app`/Dojo path; do not work around the cast by
  injecting a raw Aqua poke event.

### A callback is rejected

- Confirm the observer handles the exact result mark.
- Distinguish the application's reply path from Gall's effect wire.
- Confirm the callback recipient is installed and local to the guest ship.
- Register the observer expectation before triggering the operation.

### Ames requests time out despite `|hi`

- Confirm both endpoint agents are running.
- Check the app's configured seeds or peer identities.
- Validate node-ID-to-ship conversion and the expected sender.
- Use targeted app logs to see request and response, not host-side polling.

### PKI, breach, or key-change behavior looks unrealistically simple

- For ordinary Azimuth identities, confirm `start-azimuth` and `spawn`.
- For Groundwire identities, confirm the non-fake provider fixture and required
  Jael source updates; `start-simple` is valid for a Groundwire-only topology.
- Do not use `(init-ship who &)` for behavior that depends on Jael's view of
  Azimuth, key life, rift, sponsorship, or continuity.
- After a breach, reinitialize the breached ship without spawning it again.

### CPU or memory spikes around `<<behn>>`

- Search for raw `send-events` pokes carrying a `vase` or `!>` result.
- Search for loops containing `sleep`, `%wait`, or repeated Aqua scries.
- Remove timer-based readiness and completion polling.
- Reduce oversized logs and effects crossing the virtual boundary.
- Set shorter application timeouts for tests.
- Ensure a completed operation cancels or consumes its own pending timers.

### A Gall scry fails unexpectedly

- Append the return mark to a `%gx` path, commonly `/noun`.
- Prefer typed callback notification when waiting for a state transition.
- Treat remote scry as a read, not a completion/subscription mechanism.

### A restored fleet stalls or reports obsolete Ames bones

- Check guest readiness after restore, not only the Aqua poke acknowledgement.
- Confirm one helper set is active for the runner, not one accumulated per test.
- Pause the previous fleet and account for deferred helper effects before
  replacing its flows.
- On Groundwire, verify source registration and the required directory/key
  entries after restore; do not assume all live Jael source state survived.

## Test quality

A useful Aqua test proves behavior unavailable to a pure unit test: ship boot,
Ames delivery, Gall mark conversion, timeout handling, callbacks, or
multi-agent composition. Keep pure selection and state-transition cases in
ordinary Hoon unit tests; this makes the integration thread smaller, faster,
and easier to diagnose.

Publish each tested message once and verify every intended receiver. Repeated
publication can conceal loss of the original event. Install all receivers'
observers before publishing, and retain facts for other pending receivers
while awaiting one; sequential waits must not discard an early delivery.
For negative assertions,
prefer an explicit rejection acknowledgement or a later event that causally
fences the same path; an immediate absence scry after an asynchronous poke is
not proof of non-delivery. A positive event on an unrelated recipient is not
automatically a fence for every rejected subscription.

### Suite-level correctness

An aggregate runner must account for every selected test, not merely the ones
that compiled and reached an assertion:

- Return overall failure for any selected file/arm build failure, setup
  failure, assertion failure, or timeout. Report skips explicitly rather than
  counting them as passes; an empty selection is not evidence of coverage.
- Bound each case, including restore/readiness setup and observation waits.
  Keep build/setup errors distinct from assertion failures and timeouts so a
  broken fixture is not mistaken for an application regression.
- On failure or timeout, clean up the case's watches and child work using the
  inspected Spider lifecycle, then pause/isolate its fleet before continuing.
  If safe isolation cannot be established, fail the suite rather than running
  later cases against contaminated state. Cleanup must not erase the original
  failure or turn it into success.
- Use the smallest topology that proves each scenario and separate snapshots
  where topology requirements differ. Restore a known baseline and perform
  fresh readiness checks for each case; do not depend on a previous test's
  successful mutations. See [helper and snapshot lifecycle](#helper-and-snapshot-lifecycle).
- Establish only necessary directed routes, including provider-required
  sponsor contact. Do not add an all-to-all handshake matrix as a generic
  readiness fence; see the interaction guidance above.

The single-publication and causal-negative-assertion rules above apply to
every case, regardless of whether it passes when run alone or in the suite.
