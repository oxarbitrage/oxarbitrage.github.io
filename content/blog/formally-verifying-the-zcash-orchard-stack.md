+++
title = "Formally Verifying the Zcash Orchard Cryptographic Stack"
date = 2026-05-04
template = "page.html"
description = "A layer-by-layer account of machine-verified proofs for the Pasta curves, Poseidon hash, Sinsemilla, RedPallas signatures, Orchard protocol, and Halo 2 gadgets — with an honest inventory of what was proven and what remains axiomatized."
+++

Formal verification in cryptography is not new. Proofs of security reductions, game-hopping arguments, computational assumptions — these have been part of the discipline since the early days of provable security. What *is* relatively new is machine-checked verification: proofs that a theorem prover has checked, line by line, against an explicit formal model.

This post describes an attempt to formally verify the cryptographic stack underlying Zcash's Orchard shielded protocol, from the elliptic curves at the base to the Halo 2 gadgets at the top. The work spans six Lean 4 repositories, each covering one layer of the stack, all building on Mathlib4. Every claimed theorem has been checked by the Lean kernel. No `sorry` statements remain in any of them.

The goal here is not to survey what formal verification *can* do in principle. It's to describe what was actually proven, what was left as an axiom and why, and what the resulting theorems mean cryptographically.

---

## The Stack

The Orchard shielded pool is built from a precise set of cryptographic primitives:

- **Pasta curves** — two elliptic curves (Pallas and Vesta) whose group orders and field characteristics form a 2-cycle.
- **Poseidon** — a ZK-friendly hash function over the Pallas base field, used for nullifier derivation.
- **Sinsemilla** — a hash-to-curve construction over Pallas, used for note and Merkle commitments.
- **RedPallas** — a re-randomizable signature scheme (RedDSA over Pallas), used for spend authorization and value binding.
- **Orchard protocol** — the shielded protocol composing the above into value commitments, nullifiers, note commitments, and action circuits.
- **Halo 2** — the PLONKish proving system with IPA polynomial commitments, whose circuit gadgets implement the Orchard primitives.

The dependency chain is strict: `pasta-formal` at the base, then the three primitives in parallel, then `orchard-formal` composing them, with `halo2-formal` sitting alongside and drawing from the primitives directly.

```
pasta-formal
     │
     ├─── poseidon-formal
     ├─── sinsemilla-formal
     └─── redpallas-formal
                │
          orchard-formal
                │
          halo2-formal (also imports pasta, poseidon, sinsemilla)
```

Each repository is self-contained: it compiles independently, its claims are stated and proven within the repository, and downstream repositories take its results as imported theorems.

---

## Layer 1: Pasta Curves

The Pallas and Vesta curves are defined by the same short Weierstrass equation, *y*² = *x*³ + 5, but over different prime fields:

- **Pallas** over 𝔽_p, where p ≈ 2²⁵⁴ (the Pallas base field, also the Vesta scalar field)
- **Vesta** over 𝔽_q, where q ≈ 2²⁵⁴ (the Vesta base field, also the Pallas scalar field)

The design is deliberate: the group order of Pallas equals q, and the group order of Vesta equals p. This 2-cycle is what enables efficient recursive proof composition in Halo 2: a Pallas circuit can verify a Vesta proof, and vice versa, with no field extension mismatch. It also means that scalar multiplication on Pallas lives in the same field as arithmetic on Vesta — the two layers fit together without conversion.

### What Was Proven

The `pasta-formal` repository proves:

1. **Primality of p and q** — both ~254-bit primes are proven prime using the Lucas primality test with hierarchical Pratt certificates. The certificate chain for each prime unfolds through 28 intermediate primes across 4 levels, bottoming out at small primes verifiable by `norm_num`. This is fully constructive: no external oracle was consulted.

2. **Both curves are elliptic** — the discriminant Δ = −10800 is a unit in both 𝔽_p and 𝔽_q, i.e., 10800 ≢ 0 (mod p) and 10800 ≢ 0 (mod q). This gives `IsElliptic` instances for both curves.

3. **Same equation** — `Pallas.a₁ = Vesta.a₁`, etc., proven by `rfl`.

### The Open Problem: Group Orders

The 2-cycle property — |Pallas(𝔽_p)| = q and |Vesta(𝔽_q)| = p — is stated in `pasta-formal/Pasta/Cycle.lean` as axioms:

```lean
axiom cycle_conjecture_pallas : Nat.card (Pallas.toAffine.Point) = Fq.p
axiom cycle_conjecture_vesta  : Nat.card (Vesta.toAffine.Point) = Fp.p
```

These are not proven, and the gap is real. Proving the group order of an elliptic curve over a prime field requires either Schoof's algorithm (polynomial time but non-trivial to formalize), an SEA variant, or a certified AGM computation — none of which are currently available in Mathlib. The same result is used as `order_pallas` in `redpallas-formal`. Naming it explicitly in both places makes the dependency visible.

### 2-Adicity

Both p−1 and q−1 are divisible by 2³². This is the 2-adicity property that enables NTT-based polynomial arithmetic — the fast Fourier transform over 𝔽_p works at depth 32. It is a consequence of the primality factorizations:

- p − 1 = 2³² × 3 × 463 × 539204044132271846773 × 8999194758858563409123804352480028797519453
- q − 1 = 2³² × 3² × 1709 × 24859 × 1690502597179744445941507 × 10427374428728808478656897599072717

These factorizations are the Pratt certificates.

---

## Layer 2: Poseidon — A ZK-Friendly Permutation

Poseidon is an algebraic hash function designed specifically for ZK circuits. Its key design principle: minimise the number of field multiplications per output bit. This is achieved by using a power-map S-box (x ↦ xᵅ) rather than a Boolean circuit — a single squaring in 𝔽_p costs one constraint, versus 17+ for a SHA-256 round.

The Orchard instantiation uses:
- State width t = 3 (elements of 𝔽_p)
- R_F = 8 full rounds (S-box on all 3 elements)
- R_P = 56 partial rounds (S-box on first element only)
- α = 5 (the S-box exponent)
- A 3×3 Cauchy MDS matrix over 𝔽_p

### Why α = 5?

The S-box x ↦ x⁵ is a permutation of 𝔽_p if and only if gcd(5, p−1) = 1. For the Pallas field, p−1 has no factor of 5 (as one can verify from the Pratt certificate above). This is formally proven:

```lean
theorem alpha_coprime : Nat.Coprime 5 (Fp.p - 1)
```

verified by `native_decide`. The common alternative α = 3 is not usable here: gcd(3, p−1) ≠ 1 for the Pallas field (p−1 is divisible by 3).

### The Headline Theorem: Permutation Bijectivity

> **Theorem `permutation_bijective`**: The full Poseidon permutation — 4 full rounds, 56 partial rounds, 4 full rounds — is a bijection on 𝔽_p³.

The proof is layered:

1. `sbox_bijective` — x ↦ x⁵ is a bijection on 𝔽_p, via Fermat's little theorem: x^(5d) = x where 5d ≡ 1 (mod p−1).
2. `mds_mul_inv` — the MDS matrix has an explicit inverse, verified by `native_decide`.
3. `mixLayer_bijective`, `fullRound_bijective`, `partialRound_bijective` — each round is a composition of bijections.
4. `applyRounds_bijective` — applying bijective rounds preserves bijectivity (induction on round count).
5. `permutation_bijective` — follows immediately.

**What this gives cryptographically:** A bijective permutation is the core requirement for the sponge construction's indifferentiability from a random oracle. If the permutation were not bijective, the sponge could have fixed points or cycles that would break the collision-resistance argument.

**What this does not give:** Preimage resistance and collision resistance are not proven. They are conjectured from the algebraic degree growth across rounds — the effective degree of the composition grows exponentially, making algebraic attacks infeasible in practice. Formalizing this argument would require substantially more machinery.

The 192 round constants (64 rounds × 3 state elements) are instantiated as concrete 𝔽_p values generated by Grain LFSR, cross-checked against the `pasta-hadeshash` Sage reference. There are no axioms in `poseidon-formal` — bijectivity is proven from first principles.

---

## Layer 3: Sinsemilla — Hash-to-Curve with a Security Proof

Sinsemilla is designed for use inside arithmetic circuits, specifically for hashing the contents of Orchard notes into Merkle tree commitments. It operates by accumulating elliptic curve points, making it efficient to verify in a PLONKish circuit.

The construction: given domain separator D and bit message M, split M into n chunks of k = 10 bits each. Let Q(D) = GroupHash("z.cash:SinsemillaQ", D) and S(j) = GroupHash("z.cash:SinsemillaS", j) for j ∈ {0, ..., 1023}. Define the accumulator:

$$\text{Acc}_0 = Q(D), \quad \text{Acc}_{i+1} = [2] \cdot \text{Acc}_i + S(m_i)$$

The hash output is the x-coordinate of Acc_n.

### The Headline Theorem: Pedersen Equivalence

> **Theorem `hashToPoint_pedersen`**: Let n = |pad(M)|. Then
> $$\text{hashToPoint}(D, M) = [2^n] \cdot Q(D) + \sum_{i=0}^{n-1} 2^{n-1-i} \cdot S(\text{pad}(M)_i)$$

This says Sinsemilla computes a *Pedersen vector commitment* — a weighted sum of the generators S(j) with coefficients determined by the message chunks. The proof goes through `step_eq_double_add` (the accumulator step is `[2]·P + S(mᵢ)`) and then `foldl_step_pedersen` (unrolling the fold).

The significance: the Pedersen structure connects Sinsemilla to the discrete logarithm problem. The right-hand side is an algebraic combination of the generators S(j), and for this to be zero the message chunks must form a DLP relation.

### The Security Reduction

Two further theorems make this precise.

First, the χ (chi) function: χ(m)_j = Σᵢ 2^(n−1−i) · δ(mᵢ, j) maps a chunk sequence to a coefficient vector over ℕ. The weight of generator S(j) in the Pedersen sum is χ(m)_j.

> **Theorem `chi_injective`**: For equal-length chunk sequences m₁ ≠ m₂, the coefficient vectors χ(m₁) and χ(m₂) differ.

This is a combinatorial result — the binary encoding ensures distinct sequences produce distinct weight vectors.

> **Theorem `collision_implies_sumChunks_eq`**: If hashToPoint(D, M₁) = hashToPoint(D, M₂) with |pad(M₁)| = |pad(M₂)|, then Σⱼ (χ(m₁)_j − χ(m₂)_j) · S(j) = 0.

Combined: a Sinsemilla collision on equal-length messages yields a non-trivial linear combination of the S(j) generators that evaluates to zero — a discrete logarithm relation on Pallas. Under DLP hardness, such a relation cannot be found efficiently.

The length constraint is essential: Sinsemilla is not designed to be collision-resistant across different message lengths. The Orchard circuit enforces a fixed message length, so this is not a practical limitation.

**Axiom:** `groupHash_ne_zero` — the GroupHash function always produces non-identity points. This cannot be proven without formalizing the SWU map + BLAKE2b pipeline, which would be a substantial separate project.

---

## Layer 4: RedPallas — Re-randomizable Signatures

RedPallas is a RedDSA instantiation over the Pallas curve. It is used for two purposes in Orchard: SpendAuthSig (authorizing note spends) and BindingSig (proving value balance). The re-randomization property is what makes it suitable for both: the same key can be used multiple times without linking spends together.

The scheme:
- **keygen(sk)**: vk = [sk]·G
- **sign(sk, msg, r)**: R = [r]·G, c = H(R, vk, msg), S = r + c·sk (mod q)
- **verify(vk, msg, σ)**: check [S]·G = R + [c]·vk
- **rerandomizeKey(α, vk)**: rk = vk + [α]·G
- **rerandomizeSk(α, sk)**: rsk = sk + α

### The Headline Theorem: Re-Randomization Correctness

> **Theorem `verify_rerandomized`**: For any sk, α, msg, r, a signature produced by sign(sk+α, msg, r) verifies against vk + [α]·G.

The proof is purely algebraic: [r + c·(sk+α)]·G = [r]·G + [c·sk]·G + [c·α]·G = R + [c]·vk + [c·α]·G = R + [c]·(vk + [α]·G).

Two homomorphism results are proven. `keygen_add` — keygen(a + b) = keygen(a) + keygen(b) — makes keygen a group homomorphism from (𝔽_q, +) to (Pallas, +). `keygen_mul` — keygen(a · b) = a ⬝ keygen(b) — shows the stronger module homomorphism property: scaling the secret key is the same as scalar-multiplying the public key. Together they establish that keygen is an 𝔽_q-module map, the full algebraic structure expected of a linear key derivation.

### What Is Not Formalized

Existential unforgeability (EUF-CMA) is not proven, and the design does not claim to prove it here. Unforgeability requires modeling H as a random oracle and showing that constructing a valid signature without knowledge of sk is as hard as solving DLP. The ROM framework is not yet formalized in Lean/Mathlib at the level needed. The correctness results here are purely algebraic.

**Key axiom:** `order_pallas` — the Pallas group has prime order Fq.p. This is the same open result as `cycle_conjecture_pallas` in `pasta-formal`, and it is needed here to establish that scalar multiplication by an element of 𝔽_q is well-defined.

---

## Layer 5: Orchard — The Protocol

`orchard-formal` composes the three primitives into the Orchard shielded protocol. The formalization covers the algebraic core: value commitments, nullifiers, note commitments, DH key agreement, Merkle authentication paths, and action circuit soundness. Roughly 35 theorems are proven.

### Value Commitment Homomorphism

Value commitments take the form cv = [v]·V + [rcv]·BindingG, where V and BindingG are generators, v ∈ 𝔽_q is the value, and rcv ∈ 𝔽_q is the randomness. The key algebraic property:

> **Theorem `valueCommit_add`**: valueCommit(v₁ + v₂, r₁ + r₂) = valueCommit(v₁, r₁) + valueCommit(v₂, r₂).

This additive homomorphism is what allows a multi-action transaction to verify balance with a single binding signature: the sum of value commitments for balanced actions collapses to [rcv_net]·BindingG, and a RedDSA signature on this is a proof of knowledge of rcv_net.

> **Theorem `balance_multi_binding`**: For n actions with total value 0 (balanced), the net binding verification key equals [Σᵢ rcvᵢ]·BindingG.

### The Headline Theorem: Nullifier Uniqueness

Nullifiers in Orchard are derived as nf = [Poseidon(nk, ρ) + ψ]·K + cm, where nk is the nullifier key, ρ and ψ are note-specific values, and cm is the note commitment. Spending a note reveals nf but not the note itself.

> **Theorem `nullifier_uniqueness`**: If two notes produce the same nullifier (under the same nk), they have the same note commitment.

This is the formal proof of double-spend prevention: the nullifier set is injective over notes (given the same key). The proof uses the linear independence of K and cm via a DLP argument (`nullifier_psi_collision`: distinct ψ values give distinct nullifiers unless K and cm are DLP-related).

### DH Key Agreement

The Orchard encryption scheme uses a Diffie-Hellman construction. The sender computes the shared secret as [esk]·pk_d, and the recipient computes [ivk]·epk, where epk = [esk]·g_d and pk_d = [ivk]·g_d.

> **Theorem `dh_shared_secret`**: [esk]·([ivk]·g_d) = [ivk]·([esk]·g_d).

This is scalar multiplication commutativity, proven from the bilinearity of scalar multiplication over the Pallas group.

### Merkle Tree Security

The Merkle tree uses Sinsemilla hash for internal nodes. The formalization proves:

> **Theorem `node_root_injective`**: Under collision resistance of the Merkle hash, equal roots imply equal left and right subtree roots.

This is stated with an explicit `merkleHash_collision_resistant` axiom — a standard assumption rather than a derived result.

### The Composite Result

The `spend_authorization` theorem brings together: balanced value commitments, valid binding signature, and nullifier uniqueness into a single composite guarantee. It is the algebraic core of Orchard's privacy and integrity properties.

---

## Layer 6: Halo 2 — The Proof System

`halo2-formal` covers the PLONKish constraint system and the circuit gadgets that implement Orchard primitives. It is distinct from the other repositories in that it formalizes the *proof system mechanics* rather than the *protocol semantics*.

### PLONKish Arithmetization

A Halo 2 circuit is a table of field elements, constrained by:
- **Gates**: polynomial relations among cells in the same row. Proven:

  > **Theorem `addGate_sound`**: A satisfied addition gate implies a + b = c.
  > **Theorem `boolGate_sound`**: A satisfied Boolean gate implies a ∈ {0, 1}.

- **Permutation argument**: copy constraints (equal cells in different positions) are enforced by a grand product check. The `permCheck_sound` axiom is the Schwartz-Zippel soundness claim (discussed below).

- **Lookup argument**: cells can be constrained to lie in a lookup table. `lookupSatisfied_of_witness` and its converse are both proven.

### Range Check

The range check gadget decomposes a value into w words of base b:

> **Theorem `range_check_soundness`**: A valid base-b decomposition of width w implies v < b^w.  
> **Theorem `range_check_completeness`**: If v < b^w, a valid decomposition exists.

Both are proven via Euclidean division — a clean mathematical result with no axioms.

### ECC Gadgets

Two scalar multiplication circuits are formalized:

- **Fixed-base** (3-bit windows, 85 windows): each window selects from 8 precomputed points; the result is assembled from window outputs.

  > **Theorem `fixedBaseMul_eq_smul`**: The windowed computation equals standard scalar multiplication.

- **Variable-base** (bit decomposition, double-and-add):

  > **Theorem `variableBaseMul_eq_smul`**: The bit-by-bit computation equals standard scalar multiplication.

These are the circuit-to-specification equivalence theorems for EC scalar multiplication.

### The Headline Theorem: Sinsemilla Circuit = Protocol

> **Theorem `sinsemilla_circuit_pedersen`**: The Sinsemilla circuit gadget output equals [2^n]·Q(D) + Σᵢ 2^(n-1-i)·S(mᵢ).

This connects the circuit layer (`halo2-formal`) to the protocol layer (`sinsemilla-formal`): the Halo 2 gadget computes exactly what the Sinsemilla specification says it should. Combined with `hashToPoint_pedersen` from `sinsemilla-formal`, the full chain from circuit to cryptographic definition is closed.

### Fiat-Shamir Transcript

The transcript uses Poseidon as the hash:

> **Theorem `transcript_separation`**: Different domain separator OR different absorbed value → different Fiat-Shamir challenge.

This is the binding property. The `poseidon_collision_resistant` axiom is needed — bijectivity alone does not imply collision resistance.

---

## What Was Proven vs. What Was Assumed

The following table gives a complete inventory of axioms across all six repositories, with cryptographic justification for each.

| Axiom | Repository | Type | Why axiomatized |
|-------|-----------|------|-----------------|
| `cycle_conjecture_pallas` | pasta-formal | Group theory | \|Pallas(𝔽_p)\| = q — requires Schoof's algorithm |
| `cycle_conjecture_vesta` | pasta-formal | Group theory | \|Vesta(𝔽_q)\| = p — same |
| `groupHash_ne_zero` | sinsemilla-formal | Hash-to-curve | Full SWU + BLAKE2b formalization needed |
| `order_pallas` | redpallas-formal | Group theory | Same as `cycle_conjecture_pallas` |
| `challengeHash` | redpallas-formal | Random oracle | BLAKE2b, opaque by design |
| `G_ne_zero` | redpallas-formal | Generator | Certified Zcash parameter |
| `ValueBaseV`, `K`, `NoteCommitR` | orchard-formal | Generators | Certified parameters |
| `noteCommitHash` | orchard-formal | Hash function | Commitment hash, axiomatized as opaque |
| `merkleHash_collision_resistant` | orchard-formal | Hardness | Standard collision-resistance assumption |
| `commitToLeaf` | orchard-formal | Leaf hash | Axiomatized as opaque |
| `ipaCompleteness` | halo2-formal | IPA soundness | Inner Product Argument — deep proof theory |
| `permCheck_sound` | halo2-formal | Schwartz-Zippel | See below |
| `poseidon_collision_resistant` | halo2-formal | Hardness | Not derivable from bijectivity |
| `add_ne_zero_of_nonexceptional` | halo2-formal | DLP | Requires group order axiom |
| `G`, `H` | halo2-formal | Generators | Pedersen commitment generators |

A few axioms deserve special discussion.

**`order_pallas` / `cycle_conjecture_pallas`**: These appear in two repositories under different names but state the same thing. Consolidating them under the pasta-formal axiom is the right long-term fix. The proof strategy is clear (Schoof's algorithm), and the result is numerically verified by the Zcash team using Sage scripts — it is just not yet machine-checked.

**`permCheck_sound`** (Schwartz-Zippel): The permutation argument soundness says: if the grand product check passes, the copy constraints are satisfied. The argument is probabilistic — for a fixed failing witness, at most n values of the challenge β can make the check pass (where n is the circuit width). Since |𝔽_p| ≈ 2²⁵⁴, the soundness error is negligible. A formal proof would require: (1) expressing permCheck as a rational function evaluation, (2) showing the numerator polynomial is nonzero when copy constraints fail, (3) applying `Polynomial.card_roots_le_degree` from Mathlib. All three steps are in principle available in Lean; they require substantial setup.

**`poseidon_collision_resistant`**: Poseidon's collision resistance is conjectured from algebraic degree bounds across rounds — the effective degree grows exponentially, reaching approximately (t · R_F/2 + R_P)^t after all rounds. Formalizing this argument would require symbolic computation of polynomial degree through the permutation, which is a research-level Lean project.

**`ipaCompleteness`**: The Inner Product Argument is a complete, sound interactive proof system for evaluations of committed polynomials. Completeness (honest prover succeeds) follows from the IPA protocol definition; soundness (cheating prover fails) requires rewinding arguments. Neither is in Mathlib. This is an open problem in formal verification of SNARKs generally.

---

## Reflections on Lean 4 for Cryptographic Verification

**What worked well**: Mathlib's algebraic hierarchy (elliptic curves, finite fields, polynomial rings) is mature enough to state and prove the theorems here without building from scratch. The Lucas primality test, Fermat's little theorem, `ZMod`, `WeierstrassCurve`, and `native_decide` for computational checks — these are all available and well-supported.

**What required workarounds**: Scalar multiplication with scalars from a different field (e.g., 𝔽_q-valued scalars on the Pallas group) is not in Mathlib's type class hierarchy. `redpallas-formal` defines `fqSmul` as an explicit function and proves its linear algebra manually. This is sound but requires care to avoid naming clashes with the existing `SMul` instances.

**What would require new Mathlib contributions**:
- Schoof's algorithm or an AGM certificate checker — to close the group order gap.
- Schwartz-Zippel with symbolic polynomial representations — to prove the permutation argument soundness.
- The ROM (random oracle model) framework — to formalize EUF-CMA security of RedPallas.
- IPA completeness and soundness — a substantial stand-alone project.

The six repositories together cover the algebraic and protocol-level correctness of Orchard's cryptographic stack. They do not cover probabilistic security (unforgeability, collision resistance) or the full SNARK security of Halo 2. The boundary between what is and is not proven is, as far as possible, made explicit through the axiom inventory above.

---

## Source

All six repositories are open source:

- [pasta-formal](https://github.com/oxarbitrage/pasta-formal) — Pasta curves (Pallas and Vesta)
- [sinsemilla-formal](https://github.com/oxarbitrage/sinsemilla-formal) — Sinsemilla hash
- [redpallas-formal](https://github.com/oxarbitrage/redpallas-formal) — RedPallas signatures
- [poseidon-formal](https://github.com/oxarbitrage/poseidon-formal) — Poseidon hash
- [orchard-formal](https://github.com/oxarbitrage/orchard-formal) — Orchard protocol
- [halo2-formal](https://github.com/oxarbitrage/halo2-formal) — Halo 2 gadgets
