# Azimuth-backed Aqua ships

Read this reference when the behavior under test depends on real simulated PKI
rather than only packet delivery between userspace agents.

This reference describes ordinary Urbit's simulated Azimuth provider, not all
non-fake setups. For Groundwire-backed identities, use
[groundwire.md](groundwire.md) instead. A mixed Groundwire/Azimuth topology may
need both providers, but a Groundwire-only topology need not start Azimuth.

## Choosing ship and PKI modes

Keep identity boot mode separate from the source of identity information:

| Choice | When appropriate | Setup |
| --- | --- | --- |
| Fake ships | Userspace protocol tests needing delivery, not real PKI decisions | `start-simple`, then `(init-ship who &)` |
| Simulated Azimuth identities | Testing Azimuth-derived identity state and transitions | `start-azimuth`, `spawn` each Azimuth identity, then `(init-ship who \|)` |
| Other non-fake PKI fixtures | Testing a different identity provider, such as Groundwire | Provider-specific non-fake boot and identity updates; Groundwire-only comets use `start-simple`, `start-gw-comet`, and Jael udiffs |

`start-simple` does not make guests fake, and non-fake boot does not imply
Azimuth provisioning. Choose the provider whose decisions the test observes.
Groundwire setup details are isolated in its reference and should be ignored
for regular Urbit work. Mixed-provider tests may need both provisioning paths.
Moons and ordinary non-fake comets have specialized boot paths described below;
do not apply the Azimuth planet `spawn` sequence to every identity class.

## Choosing Azimuth-backed ships

Use Azimuth-backed ships when testing Azimuth-derived behavior involving:

- Ames or Jael kernel behavior;
- `%base` code that consumes identity or networking state;
- ownership and sponsorship relationships;
- public-key lookup, key lives, or key rotation;
- continuity breaches and rift changes;
- how peers reject old keys or recover after a breach; or
- network-core behavior whose decisions depend on Azimuth proofs.

Fake ships are appropriate when the assertion is confined to a userspace
protocol and only needs working Ames delivery. They deliberately bypass parts
of real identity provisioning, so a passing fake-ship test is not evidence that
PKI lifecycle behavior works.

## Basic simulated Azimuth setup

The current Arvo `ph-io` pattern is:

```hoon
/-  spider
/+  *ph-io
=,  strand=strand:spider
^-  thread:spider
|=  vase
=/  m  (strand ,vase)
;<  ~  bind:m  start-azimuth
;<  ~  bind:m  (spawn ~bud)
;<  ~  bind:m  (spawn ~dev)
;<  ~  bind:m  (init-ship ~bud |)
;<  ~  bind:m  (init-ship ~dev |)
;<  ~  bind:m  (send-hi ~bud ~dev)
;<  ~  bind:m  end
(pure:m *vase)
```

`start-azimuth` starts the ordinary Aqua helpers and initializes Aqua's
Azimuth simulation. `spawn` creates the identity, keys, and simulated Azimuth
logs. The `|` argument to `init-ship` means non-fake; `&` means fake.

Spawn an Azimuth identity once before its first non-fake boot. When one scenario
boots the same identity again—for example after a breach—do not spawn it again.
The point is to preserve and evolve the simulated identity history. This rule
does not require spawning comets or identities backed by a different provider.

Provision all required identities before relying on their PKI relationships.
For galaxies, stars, and planets, inspect current Arvo tests for any hierarchy-
specific ordering or sponsor setup instead of assuming the arrangement is
irrelevant.

## Continuity breach pattern

A breach rotates identity state and advances continuity. A typical peer-aware
test is:

```hoon
;<  ~  bind:m  start-azimuth
;<  ~  bind:m  (spawn ~bud)
;<  ~  bind:m  (spawn ~dev)
;<  ~  bind:m  (init-ship ~bud |)
;<  ~  bind:m  (init-ship ~dev |)
;<  ~  bind:m  (send-hi ~bud ~dev)
;<  ~  bind:m  (breach-and-hear ~dev ~bud)
;<  ~  bind:m  (send-hi-not-responding ~bud ~dev)
;<  ~  bind:m  (init-ship ~dev |)
;<  ~  bind:m  (send-hi ~bud ~dev)
```

`breach-and-hear` breaches its first ship and waits until the second ship has
observed the new rift. Use it when peer awareness is part of the causal setup.
Use `breach` when no particular peer must first observe the change. Afterward,
reinitialize the breached identity as non-fake so it boots with the new keys.

The optional `send-hi-not-responding` assertion demonstrates that the old
instance/key state no longer establishes communication before the new ship is
initialized. Include it only when that transition is relevant to the test.

For multiple breaches, preserve the same rule: breach, allow required peers to
observe it, reinitialize the affected ship, and never respawn the identity just
to obtain new keys.

## Moons and comets

Moons and comets use specialized `ph-io` paths:

- For a non-fake moon, provision its sponsor, then use `init-moon moon |` so
  the sponsor creates the moon keys before Aqua boots it. Inspect the current
  implementation's supported moon forms.
- Comets are not Azimuth-spawned identities. Use `init-comet` and the current
  test fixture/key behavior rather than `spawn`. Some ordinary Urbit versions
  have an `init-comet` hardcoded to one mined comet/feed; do not assume it accepts
  any arbitrary comet just because its argument mold is `ship`.

Do not generalize the planet `spawn`/`init-ship` sequence to these identity
classes without checking the current `ph-io` source.

## What to validate

Synchronize on the behavior that proves the PKI transition, such as:

- the peer observing a new rift;
- traffic under old continuity failing;
- the reinitialized ship communicating with its new keys;
- Jael returning the expected life/public key; or
- Ames changing its peer state after receiving the simulated Azimuth update.

Prefer existing `ph-io` helpers such as `breach-and-hear` over timer guesses or
host-side scry polling. Keep application interactions on `poke-app`/Dojo paths;
using non-fake ships does not make raw Aqua poke events safe.

Inspect those helpers before estimating runtime: existing PKI helpers such as
`breach-and-hear` may contain sleeps and scry polling internally. Prefer their
established sequencing, but do not copy their polling into new application
readiness tests or claim that they are event-only observers.

## Inspect current examples

Before implementing, search the checked-out Urbit source under `/ted/ph` for:

- `hi-az.hoon` for basic non-fake communication;
- `breach-hi.hoon`, `breach-multiple.hoon`, and `breach-sync.hoon` for breach
  sequencing;
- `moon-az.hoon` for moons;
- `hi-comet-az.hoon` for mixed comet/Azimuth networks; and
- `ahoy.hoon` for deeper Ames/Mesa and key-migration scenarios.

These examples and the repository's current `ph-io` are authoritative when
they differ from this reference.
