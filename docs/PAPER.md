# Constellation: Identity as a Computational Eigenform, Verified by Diverse Double Compiling

## Abstract

A system has identity when its state is publicly verifiable, privately irreproducible, and temporally
coupled. We show this is the definition of an **eigenform**: identity is the fixed point `φ(x) = x` of
the system's own validation process, the form invariant under the operation that generates it. A
Constellation node is the canonical computational instance of such a fixed point — a **quine**, a process
whose output is its own re-produced identity, re-marked every heartbeat. Its cryptographic signature is
the **eigenvalue**: the public observable by which the eigenform is recognized from outside without
exposing its private interior. We further show that a mesh of such nodes, each independently reproducing
and cross-checking the others' fixed points, is **Diverse Double Compiling (DDC)** — Wheeler's
countermeasure to Thompson's Trusting-Trust attack — lifted from one compiler-pair to a population. The
redundancy of the mesh is therefore not fault-tolerance added to the design; it *is* the trust mechanism.
Under a cooperative-drift (non-Byzantine) threat model, this yields O(1) mutual verification with instant
finality, an explicit worse-is-better trade against blockchain's O(n²) consensus. The reference
implementation ships as a standalone Go module.

## One-line thesis

If a system maintains a state that is publicly verifiable but privately irreproducible, and that state
evolves under a temporally consistent rule, then the system has identity — and that identity is the
eigenform of its own validation, verified across a mesh by diverse double compiling.

---

## 1. Identity is the eigenform of self-referential validation

Traditional systems define identity through static credentials: keys, certificates, tokens. A credential
is a thing you *have*, checked at a point in time. This is brittle in exactly the way a stolen key is
fatal — possession is identity, so theft is impersonation.

Constellation defines identity differently. **A node IS the fixed point of its own coherence
validation.** Let `F` be the operation "validate my own event ledger." A node's identity is the ledger
`x` such that `F(x) = x`: applying the validation rule to a coherent ledger leaves it unchanged. This is
not a metaphor for the implementation; it is the implementation. `coherence.go`'s `ValidateCoherence` is
`F`. It re-derives each event's hash from its canonical RFC-8785 bytes and compares computed against
stored:

```go
computed := HashEvent(canonical)
if computed != prev.Metadata.Hash { /* tampered — not at the fixed point */ }
```

A ledger at its fixed point passes all three layers (hash-chain integrity, schema, temporal
monotonicity); the validation leaves it unchanged. A tampered ledger is not at the fixed point, and `F`
reports `pass: false` on `/health`. The node is precisely the ledger invariant under its own
re-validation.

This `x = F(x)` is a **von Foerster eigenform**: the form invariant under the operation that generates it,
`φ(x) = x`. Naming it as such is not decoration — it tells us what identity *is* (a maintained fixed
point, not a possessed credential) and predicts the protocol's properties, as the following sections
develop. The April implementation stated `x = F(x)` directly and shipped it; the eigenform reading is
simply its precise account.

A system has identity, then, when its state is:

1. **Publicly verifiable.** Any peer can check the fixed point in O(1) per check (§4).
2. **Privately irreproducible.** The fixed point cannot be forged without breaking the hash chain (§5).
3. **Temporally coupled.** Recognition derives from consistent re-production of the fixed point over
   time, not from credentials held at a point in time (§3, §6).

These three criteria are the eigenform observed from outside. They are also, exactly, the lesson of the
obfuscated trojan quine: the generative interior is sealed, and identity is established by *reproduced
behavior over time*, never by reading a static representation.

## 2. The node is a quine; its signature is the eigenvalue

The simplest computational eigenform is a **quine**: a program whose output is its own source,
`φ(x) = x` where `φ` is *execute*. It is the canonical discrete fixed point — the place where eigenform
stops being a cybernetics abstraction and becomes five lines any programmer can run and verify.

A Constellation node is a quine in this exact sense. Every five seconds (`heartbeat.go:tick`) the node
appends an event to its ledger, re-commits the ledger to its in-process git store, re-derives the
tree-hash fingerprint of `events/`, and re-signs that state. **The node re-produces its own identity
every cycle.** A living distinction is one that is continuously re-marked; the heartbeat is that
re-marking. This is why a node that stops heartbeating ceases, in the constellation's eyes, to be a live
identity — it has stopped reproducing its fixed point.

The node's **ECDSA signature is the eigenvalue** of this fixed point: the public, verifiable observable
that an output belongs to this node's eigenform, without exposing the private generative interior (the
private key plus the full event history). `NodeID = SHA-256(pubkey DER)` is the public name of the
eigenform; the signature is how the eigenform makes itself recognizable from the outside. This is von
Foerster's eigen-behavior precisely: object as public token, behavior as closed recursion.

Two consequences follow directly, and both are load-bearing for the trust model:

- **Identity is in the running, not the reading.** You cannot content-hash a node's source or
  representation and call that its identity. The obfuscated trojan quine is the counterexample — two
  different source texts can share one eigenform, and one source text can behave two ways. This is *why*
  the protocol verifies reproduced behavior over time (sequence consistency, tree-hash agreement) rather
  than a static artifact. The identity lives in the fixed point being continuously reached, not in any
  frozen text.

- **The ledger is a static eigenform; the node is a dynamic one.** A sealed content-address — each
  event's `Hash`, the `NodeID` — is the **static** eigenform: `φ(x) = x` eternally, no dynamics, cannot
  drift (altered content yields a different address), substrate-invariant by construction. The
  continuously-heartbeating node process is the **dynamic** eigenform: alive, drift-tracking, paying a
  cost to hold its distinction against the pressure of partitions, time gaps, and silence. **Sealing is
  the dynamic→static transition** — freezing the alive, private, costly fixed point into a free, eternal,
  shareable public one. Each committed event is a sealing; the ledger is the accreted record of the live
  node's identity, frozen one vertebra at a time.

## 3. Diverse Double Compiling is the constellation

A single node validating its own ledger cannot, by itself, be trusted. This is Thompson's point in
*Reflections on Trusting Trust*: a compiler trojan that reproduces itself across its own fixed point is
invisible from inside its own lineage — the source audits clean while the artifact is compromised. The
deviation it carries *is* the self, faithfully reproduced. A node auditing only itself has the same
blind spot.

Wheeler's countermeasure is **Diverse Double Compiling**: compile the artifact with a second, independent
derivation and check that the two outputs match bit-for-bit. Trust comes not from reading the source but
from **independent reproduction**. Constellation implements DDC as its trust mechanism, in two
legs[^ddc-isomorphism]:

- **The double — reproduce-and-compare — is `coherence.go`.** Each node re-derives every event's hash and
  compares computed against stored ("tampered at seq N: computed != stored"). This is Wheeler's
  bit-for-bit match applied to the ledger: the node double-compiles its own history on every validation
  pass, confirming it is still at its fixed point.

- **The diverse — independent derivations — is the peer mesh.** A node cannot trust itself, but N
  independent nodes each reproducing and cross-verifying the eigenform *can* establish trust among
  themselves. Each node exchanges a signed tree-hash fingerprint with its peers and checks sequence
  consistency; trust accrues as an EMA over time (`constellation.go`). This is diverse double compiling
  lifted from one compiler-pair to a population.

The reframe this makes precise: **the constellation's redundancy is the trust mechanism, not
fault-tolerance bolted onto it.** The double (internal reproduce-and-compare) and the diverse (external
independent peers) are the two legs DDC requires. Constellation is the name for running both at
population scale. A node's identity is therefore not a key it holds but a fixed point it *is*, proven not
by authenticating a stored credential but by reproducing the fixed point — bit-for-bit on the ledger,
cycle-by-cycle across the mesh.

This is also the Trusting-Trust mitigation made positive. Thompson's compiler is a malicious quine
(untrustable because its lineage is unbootstrapped). A Constellation node is a trusted quine: its trust
comes from *pure lineage* (it bootstrapped its own ledger from a clean seed) plus *reproduction* (DDC).
Trust cannot come from the seal alone — an adversary seals too; obfuscation is sealing's dark mirror,
opaque interior with observable behavior. Trust comes from the lineage and the reproduction, which is why
verification reproduces behavior over time rather than auditing a representation.

[^ddc-isomorphism]: The claim that Constellation's peer mesh *is* DDC is a structural-isomorphism claim,
not a rigorous identity. Wheeler's DDC verifies a **compiler binary** — two independent compilations of
the same source; the outputs must match bit-for-bit. Constellation's "double" verifies **ledger hashes** —
each node re-derives event hashes and cross-checks peers' tree-hash fingerprints. The operation is the
same (reproduce among independent derivations, trust by match), but the artifact type differs (ledger
events vs. compiled binaries). This paper treats the structural isomorphism as load-bearing for the trust
argument while marking it as a generalization of Wheeler's specific DDC rather than the original
procedure.

### EMA peer trust

Trust is tracked per peer as an exponentially-weighted moving average over heartbeat consistency:

```
trust = 0.8 · trust + 0.2 · (consistent ? 1.0 : 0.0)
```

A new peer starts neutral (0.5). Consistency means the peer's sequence advanced by exactly one since the
last heartbeat — the fixed point reproduced as expected. Trust is earned, decays on drift, and gates
whether a peer's claims are admitted:

| Level | Score | Meaning |
|-------|-------|---------|
| Trusted | ≥ 0.7 | Consistent reproduction of its fixed point over time |
| Pending | ≥ 0.4 | Insufficient history to judge |
| Suspect | ≥ 0.2 | Recent inconsistencies — possible drift |
| Rejected | < 0.2 | Persistent drift or an identity conflict |

After two consecutive drifts (`MaxDriftBeforeChallenge`), the verifier issues a **challenge**
(`protocol.go:issueChallenge`): it requests an event range from the suspect peer and re-validates the
hash chain locally — DDC's double invoked on demand against a specific peer's reproduction.

## 4. O(1) mutual verification

Verifying an eigenform via DDC does not require replaying its derivation. It requires checking that an
independent reproduction lands on the same fixed point. This is why Constellation achieves mutual
verification in O(1) per peer where blockchain consensus requires O(n) to O(n²).

On each heartbeat, a node signs `{node_id, listen_addr, tree_hash, seq, last_hash, timestamp}` and
broadcasts it. On receipt, a peer performs exactly: one signature check, one NodeID-matches-pubkey check
(preventing relay attacks), and one sequence-consistency comparison (`heartbeat.go:VerifyHeartbeat`,
`constellation.go:ProcessHeartbeat`). No event replay, no Merkle-proof traversal, no global consensus.

The tree-hash is the eigenvalue of the entire ledger: two nodes that agree on it agree on the whole
history, established in a single comparison. O(1) verification is the computational signature of "compare
fixed points," as distinct from "re-run the computation" (O(n) replay) or "agree adversarially on which
computation occurred" (O(n²) consensus). The eigenform frame is what makes the cost legible: you compare
the invariant, you do not reconstruct the trajectory.

## 5. A stolen key is insufficient for impersonation

In traditional PKI, a stolen private key is a stolen identity. In Constellation, **the key is the
eigenvalue, not the eigenform.** Holding the eigenvalue lets you sign, but the eigenform is the fixed
point reproduced over time — the behavioral history — and that cannot be forged.

When an attacker who has copied a node's private key begins broadcasting, existing peers observe:

- **Sequence discontinuity.** The attacker's `seq` counter starts from 1; peers expect `last_known + 1`.
  The attacker cannot reproduce the victim's history because the hash chain is computationally
  irreversible.
- **Identity conflict.** Two different addresses claim the same NodeID within the 30-second
  `IdentityConflictWindow`. `ProcessHeartbeat` rejects both and sets trust to 0.
- **Trust collapse.** The EMA trust score drops below the rejection threshold.

The attacker holds the eigenvalue but cannot occupy the victim's eigenform; the best it can do is start a
new, conflicting eigenform under a colliding name, which the conflict detector catches. Trust is coupled
to history, not credentials. This distinguishes Constellation's model from a static-credential PKI: a
credential alone is not a vouchable identity; the identity is the credential plus a verifiable,
DDC-reproducible behavioral history. (`scenario_theft.sh` exercises this end-to-end.)

## 6. The threat model: cooperative drift, by deliberate choice

DDC's threat model is to catch *drift and inserted tampering* by reproduction among cooperating
derivations — not to defeat a total adversary. This is precisely Constellation's posture. It assumes
cooperative agents that may drift but do not actively deceive:

| Property | Blockchain | Constellation |
|----------|-----------|--------------|
| Consensus cost | O(n) to O(n²) | O(1) per peer |
| Finality | Probabilistic or expensive | Instant (coherence = final) |
| Scaling | Each node increases cost for all | Each node only increases local cost |
| Threat model | Adversarial (Byzantine) | Cooperative (drift, not deception) |
| Sybil resistance | PoW/PoS (energy/capital) | Not needed |

This is a worse-is-better trade made deliberately and stated as a virtue, not an apology. Blockchain pays
the O(n²) Byzantine cost because its threat model is adversarial. Constellation declines that cost — the
New-Jersey choice — because its regime is a mesh of cooperating nodes (a CogOS workspace hierarchy) where
the real failure mode is drift, not betrayal. The EMA trust score and the challenge protocol are a
*drift-detector*, and the architecture is shaped around catching drift cheaply rather than defeating an
adversary expensively.

### Threat-model boundary

The drift adversary — the adversary this protocol is designed against — can cause ledger divergence and
fail silently (a node that stops heartbeating or begins producing inconsistent sequences). It cannot forge
hash-chain history or inject events into an existing ledger without detection, because every event hash is
re-derived on validation. The Byzantine adversary is categorically different: it can forge messages,
inject false events, and coordinate deception across multiple colluding nodes. Constellation declines
Byzantine resistance by deliberate design — that declination is what buys O(1) verification instead of
O(n²) consensus. The cost of the trade is concrete: a Byzantine-capable adversary with access to valid
signing keys and network control can corrupt trust scores and cause nodes to admit false peers. This paper
claims only cooperative-drift safety; any deployment where the adversary is Byzantine requires a different
protocol.

## 7. The node's anatomy: head and skeleton, joined by a membrane

A node is stratified by timescale into a slow, sealed **skeleton** and a fast, molten **head**, joined at
the membrane where they meet. This stratification is not an addition to the protocol; it is what the
protocol's two halves already are.

- **The skeleton (slow, sealed) is the committed ledger.** Each appended event is a vertebra — a
  distinction marked once and sealed as a static eigenform (a content-address that cannot drift). The
  ledger bears load by rigid axial stacking: `prior_hash` chaining, each event compressing onto all prior
  events' hashes. It is read, not rewritten. High inertia, geological timescale.

- **The head (fast, molten) is the live node process — the running quine.** It is the dynamic eigenform
  that maintains the node's identity *against pressure*: network partitions, hardware boundaries, time
  gaps. It never seals; a sealed head is a dead node. Every heartbeat is the head re-marking the
  distinction "I am still this node," re-producing-and-signing identity against the constant pressure to
  dissolve into silence. Identity is the distinction that can never finish being drawn — the moment it
  stops being re-marked it becomes a fossil.

- **The membrane is the signed heartbeat.** It is the interface where the fast head attaches to the slow
  skeleton: the heartbeat re-derives the tree-hash of the sealed ledger and signs it live, binding the
  dynamic eigenform to the static one. The skull houses the brain by enclosing it; the heartbeat houses
  the ledger by signing it — load-bearing by enclosure, not by rigidity.

This anatomy explains the continuity-across-reconnect property. If a node has not changed between
disconnect and reconnect, its activity was signed by the same authorized runtime identity, so peers
recognize it. In eigenform terms: the head-quine continued reproducing the same fixed point across the
gap, reaching the same eigenvalue (signature), so identity is continuous. Continuity-across-time *is* the
quine continuing to run.

## 8. Substrate generalization

The same fixed-point operation — mark a distinction, seal it as a content-addressed object, reconcile all
references toward it — recurs at other scales in the CogOS substrate: at the cogdoc level (memory atom
with a content-address and frontmatter refs), at the session level (live attachment with a verifiable
event chain), and at the block level (data-scale content-addressing). The architectural claim is that peer
node identity (L1, this paper's subject) is one projection of this general operation. A substrate-wide
treatment of all projections and how they compose is companion/future work; this paper develops the L1
case fully.

L2 (Identity: principal spanning many machines; OIDC-shaped) and L3 (Presence: ephemeral activation
of an L2 identity, query-derived rather than stored) are the layers immediately above L1. L1 node keys
sign L2 attestations; L3 is emergent from bus events. These projections are named here to orient the
reader; their full treatment is deferred to the companion substrate paper.

## 9. Proof structure

Each property is demonstrated by an automated test scenario; each scenario is an eigenform/DDC claim
exercised end-to-end.

| Property | What it demonstrates | Test scenario | Result |
|----------|---------------------|--------------|--------|
| Self-referential closure | `F(ledger) = ledger`: the fixed point is reached and held | `scenario_happy.sh` (trust convergence) | Pass |
| O(1) verification | compare fixed points, do not replay | `scenario_happy.sh` (heartbeat exchange) | Pass |
| Stolen keys insufficient | the eigenvalue is not the eigenform; history cannot be forged | `scenario_theft.sh` (key-theft rejection) | Pass |
| Tamper detection | the double — reproduce-and-compare — catches any deviation | `scenario_drift.sh` (self-detection) | Pass |
| Dynamic trust bootstrapping | a new quine earns recognition by reproducing its fixed point over time | `scenario_join.sh` (new node earns trust) | Pass |

## 10. Implementation

The reference implementation is complete and ships as a standalone Go module:

- ECDSA P-256 identity, `NodeID = SHA-256(pubkey DER)`; keys local, IDs derived, not registered
- Hash-chained append-only ledger, events as `events/{seq:08d}.json` in an in-process git repo (go-git);
  RFC 8785 canonical JSON, SHA-256 content hashing, tree-hash state fingerprint
- Signed heartbeats every 5 seconds; three-layer self-coherence check (hash chain, schema, temporal
  monotonicity)[^software-verified]
- EMA peer trust with configurable decay and thresholds (0.7 / 0.4 / 0.2); identity-conflict detection;
  challenge-on-drift
- Six HTTP endpoints (`/heartbeat`, `/peers`, `/challenge`, `/join`, `/health`, `/state`)
- Docker Compose infrastructure (3-node constellation + attacker); four automated test scenarios

[^software-verified]: The hash chain is verified in software: `coherence.go` re-derives each event hash
at append and on every validation walk. Nothing in this layer traps a stray write at the hardware level —
the chain is glass, not concrete. A CHERI-class hardware-enforcement layer, which would make the memory
containing event hashes unforgeable at the CPU boundary, is a buildable future direction but is not
present in the current implementation. The paper claims only what the code does.

## 11. Cross-workspace trust composition

The CogOS workspace model is hierarchical and recursive — a workspace can have an upstream the way a git
repo has a remote, with selective promotion of knowledge across layers. Constellation is the trust
substrate that makes this safe: peer nodes verify each other through hash-chained ledgers and
EMA-weighted reputation, and trust scores gate which peers can attest to which knowledge. O(1) mutual
verification plus stolen-key insufficiency are exactly the properties needed to make per-edge trust cheap
to evaluate and hard to forge — enabling recursive workspace composition without a central authority
granting permissions from above.

## 12. Relation to prior work and venues

The identity model places Constellation in conversation with second-order cybernetics (von Foerster's
eigenforms), the theory of self-reproduction (quines as the canonical discrete eigenform), and — most
directly for a security audience — the Trusting-Trust literature (Thompson 1984; Wheeler 2005, Diverse
Double Compiling). The contribution to that last thread is to recognize that a peer mesh of
self-validating nodes is DDC at population scale, and to show it shipped as a working distributed-systems
protocol rather than a one-time compiler-bootstrap exercise.

- **Distributed systems.** SOSP, OSDI, EuroSys
- **Security.** CCS, USENIX Security (positioned against Trusting-Trust / DDC)
- **Decentralized systems.** Distributed-trust venues
- **Interdisciplinary.** Complex Systems, ALIFE (the eigenform / quine / cross-substrate isomorphism)
