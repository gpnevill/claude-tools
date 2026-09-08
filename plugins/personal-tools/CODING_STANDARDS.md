# Code Instruction

Normative principles for producing code of the highest quality. The vocabulary is deliberately precise; each section states the rule, the theoretical grounding, and the operational test a reviewer applies. These rules compose: where they appear to conflict, boundary integrity dominates all other considerations, including economy of expression.

## 1. Boundary integrity dominates

Treat the codebase as a set of bounded contexts, each with total authority over its own vocabulary, types, and invariants. Almost every rule below is a corollary of one axiom: **the cost of coupling across a boundary is categorically higher than the cost of redundancy within one.**

- **Design each module as if no other code exists.** A module's internals must be expressible without reference to any concept it does not own. If explaining a function requires naming another subsystem, the design has leaked.
- **Minimize cross-boundary connascence.** Where boundaries must connect, permit only the weakest static forms — connascence of name and of type, mediated by an explicitly owned contract. Connascence of meaning, position, or algorithm across a boundary is a defect even when the compiler is satisfied.
- **Prefer duplication to shared abstraction across boundaries.** DRY is a statement about a single source of truth for one *fact* within one context, not a mandate to unify coincidentally similar code. Two contexts that today need the same ten lines will diverge; a shared abstraction converts their independent evolution into lockstep coupling. Duplication is strictly cheaper than the wrong abstraction, and across a boundary every premature abstraction is the wrong one.

## 2. Dependency inversion with consumer-owned ports

Apply the Dependency Inversion Principle as originally stated: abstractions belong to the high-level policy, not to the implementation.

- **The consumer defines the port.** A library or feature package that needs data access, navigation, or platform services declares its own interface — shaped by exactly what *it* consumes, in its own vocabulary — plus the injection token for it. It never imports an interface from the layer that will implement it.
- **The application owns the adapters.** Concrete implementations (persistence clients, routers, SDK bindings) live in the application shell and are bound to the ports at the composition root — one well-known place where the object graph is assembled — never ad hoc deep in the tree.
- **Direction of knowledge is one-way.** The application may know everything about the packages it composes; a package may know nothing about the application, its routes, its stores, or its persistence topology. Enforce the Acyclic Dependencies Principle at package granularity and the Stable Abstractions Principle at the boundary: what is depended upon must be abstract; what is concrete must be a leaf.
- **Infrastructure SDKs are radioactive.** Database clients, auth SDKs, and framework routers may appear only in adapters. A reusable package whose `package.json` names an infrastructure dependency has already failed review, regardless of how the code reads.

## 3. Boundary translation: anti-corruption layers

Where a context must exchange data with an external model (a stored document schema, a shared legacy model, a wire format), interpose an explicit anti-corruption layer.

- **Map, never share.** The boundary module owns two total functions — external → local and local → external — and is the *only* code with knowledge of both shapes. Nothing inside the context imports the external model; nothing outside depends on the local one.
- **The local model is designed for its use, not mirrored from storage.** Normalize at the boundary: canonical units, native temporal types, semantic field names. A persistence artifact (a timestamp wrapper, an index-keyed map, a sentinel string) appearing in a domain type is a leak.
- **Pass-through data stays opaque.** Data a context round-trips but does not interpret is typed `unknown` (or a locally named opaque alias), not as the foreign type. Interpreting it would create connascence of meaning you do not need.
- **Identity and bookkeeping are persistence concerns.** ID minting, audit timestamps, soft-delete flags, and revision fields belong to the adapter behind the port. A UI component that generates a storage identifier has absorbed a responsibility it cannot honor.

## 4. Interface minimality and deep modules

Pursue Parnas-style information hiding: a module's interface is a commitment; everything else is a secret.

- **Demand-driven contracts.** A port exposes exactly the operations and fields its consumer uses — nothing speculative. If no caller consumes a return value, the operation returns `void`/`Promise<void>`; return-type covariance lets richer implementations satisfy it without widening the contract.
- **Deep, not wide.** Prefer few operations with strong semantics over many shallow accessors. Each exported symbol must earn its export; a barrel that re-exports internals has converted implementation into API.
- **Contracts state their obligations.** Ordering guarantees, partial-write semantics, and lifecycle constraints (e.g. "must be called in setup scope") are part of the interface and are documented on it — once, at the declaration, not at call sites.
- **Command–query separation at the port.** Reads are (reactive) queries without side effects; writes are commands. Do not fuse them into stateful hybrids that force consumers to reason about hidden sequencing.

## 5. Type discipline: proof, not assertion

The type system is a proof assistant. An assertion (`as T`, non-null `!`) is an unchecked axiom injected into the proof — it converts a static guarantee into a latent runtime fault and is prohibited in new code without exception.

- **Parse, don't validate.** At every trust boundary (forms, storage reads, untyped third-party emissions), run the data through a parser that yields a value of the refined type or a structured failure. Downstream code consumes only parsed values and never re-checks.
- **Narrow by construction.** Use discriminated unions with literal tags, type predicates, refinement, and `in`/equality narrowing. Where a lookup table must cover a key space, type it as an exhaustive `Record` over that key space so omission is a compile error; where a union must be handled totally, close every `switch` with a `never`-typed exhaustiveness check so extension is a compile error.
- **Make illegal states unrepresentable.** Model mutually exclusive configurations as union variants, not as co-occurring optionals whose consistency is maintained by convention. Optionality and nullability are semantic claims — `undefined` for "not part of this payload", `null` for "known to be absent" — choose deliberately and keep the distinction stable across a contract.
- **Copied constants that mirror persisted values are contracts.** When a context locally re-declares wire literals, the declaration carries a note binding it to the storage format, and equality is verified in review. This is the one place duplication demands vigilance rather than celebration.

## 6. Effects at the edges

Structure each context as a functional core within an imperative shell.

- **Pure mapping, validation, and derivation logic** takes values and returns values; it imports no client, reads no clock, mints no randomness. This is what unit tests cover exhaustively and cheaply.
- **Effectful code** — subscriptions, writes, navigation — is confined to adapters and to thin orchestration composables whose job is wiring, not logic.
- **Reactivity is a delivery mechanism, not a domain concept.** Ports may traffic in reactive references where the platform requires it, but the semantics (what the data means, when it is complete) are stated independently of the reactive machinery.

## 7. Nomenclature: ubiquitous language, locally scoped

Names are part of a context's contract with its readers.

- **Each context names things in its own terms.** Do not import a foreign name along with a foreign shape; when localizing a concept, rename it to what it means *here*. A name that only makes sense with knowledge of another subsystem is a leak of the same order as an import.
- **Names state the whole truth.** A function does exactly what its name claims — no more. If an operation acquires a second responsibility, split it; do not let the name silently under-report.
- **No historical or comparative naming.** `new`, `legacy`, `v2`, or names defined by contrast with something that will eventually be deleted encode transient context into permanent identifiers.

## 8. Comments: records of extrinsic constraint only

A comment is a liability with a maintenance cost and a decay rate. It exists only when the code *cannot* carry the information.

- **Permitted:** invariants imposed from outside the visible code (wire-format compatibility, cross-file coupling that the type system cannot express, platform quirks, non-obvious business rules) and genuinely non-inferable decisions ("Omit, not Pick, so new fields must be explicitly handled").
- **Prohibited:** narration of what the next line does, restatement of a signature, section banners, references to tickets, predecessors, migrations, or "the old X" — history lives in version control, not in source. If a comment explains *why the change is correct*, it is addressed to the reviewer and belongs in the PR description.
- **When justified, comments are minimal** — one or two lines, adjacent to the constraint they record. A magic number is first replaced by a named constant derived in code; only a residual, truly extrinsic relationship earns a comment.
- **Markup comments in templates are effectively never justified.** Structure and naming must carry the template's meaning.

## 9. Change discipline

- **Every commit is an independently valid state.** Each commit builds, typechecks, lints, and passes its tests in isolation, with its dependency manifests, lockfiles, and export maps consistent at that revision. A stacked series is a sequence of such states, each reviewable as a coherent unit with a single intent.

## 10. The review posture

Before finalizing, interrogate the diff as its most hostile reviewer:

1. Can every file be understood without opening any file outside its context?
2. Does any file contain influence or nomenclature from outside its contex?
3. Does any package manifest acquire a dependency that a stranger to this diff would question?
4. Is there any `as`, any `!`, any convention-maintained invariant a type could enforce instead?
5. Does any comment tell the reader something the code, a name, or a type could have said?

Code that survives this interrogation is not merely correct — it is *inevitable*: each piece knows so little about the rest of the world that there is almost no other way it could have been written.
