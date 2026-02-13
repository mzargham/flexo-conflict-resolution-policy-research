# VCS

A **version control system** (VCS) tracks changes to a collection of artifacts over time, providing history, branching, and [[Merge|merging]]. Git is the dominant VCS for source code; [[Flexo MMS]] is a VCS for [[RDF]] graph [[Model|models]]. Both inherit a design decision that is harmless for text but consequential for structured data: the conflation of the commit *action*, the commit *artifact*, and the resulting *state* into a single noun.

---

## The Conflation

The word "commit" in Git (and most VCS) refers simultaneously to:

1. **An action** — the imperative process of recording changes
2. **An artifact** — the immutable object that results (the DAG node)
3. **A state** — the snapshot of the repository at that point

These are three distinct concepts that get collapsed into one noun. It's the classic process/product ambiguity — like how "a build" means both the act of building and the binary it produces.

---

## A Finer Decomposition

If we pull the action apart sequentially, there are at least these distinct phases:

1. **Selection** — choosing which changes to include (staging/index)
2. **Metadata assembly** — author, timestamp, parent ref(s), message
3. **Tree construction** — building the content-addressed snapshot (blobs → trees)
4. **Commit object creation** — the immutable record tying tree + metadata + parent pointer(s)
5. **Reference mutation** — moving HEAD / branch pointer to the new object
6. **Resulting repository state** — the new configuration of the DAG + working tree + index

Steps 1–5 are the *process*. Step 4's output is the *artifact*. Step 6 is the *state*. The VCS collapses 4, 5, and 6 into a single atomic concept.

---

## Why They Did It

This is a deliberate design trade-off: by making the transition and the resulting state the same addressable object (the hash), you get **referential transparency** — any reference to a commit unambiguously identifies both *what happened* and *what exists*. The hash is simultaneously a name for the artifact and an identifier for the state. That's powerful for content-addressing but it erases the process semantics entirely.

The cost is that the *action* has no first-class representation. You can't inspect or reason about the commit operation independently of its product. Things like "this commit was a rebase replay of an earlier commit" or "this commit was created by a cherry-pick" are lost or stuffed into the message string as unstructured text.

---

## Consequences for Model VCS

For text, the conflation is benign — the [[Key Insight]] explains why. A text merge is a pure function of content, application order rarely matters, and the result is self-certifying. The hash-as-identity shortcut works because there is no need to evaluate the merged state against external predicates.

For structured, constraint-laden models, the conflation breaks down. Whether a commit is acceptable depends on the state it lands on. The formalism in the [[Conflict Resolution Problem Statement]] therefore separates what VCS collapses: commits become *control inputs* ($u \in \mathcal{U}$), model states become *state variables* ($X \in \mathcal{X}$), and the state transition function $f(X, u) = X^+$ connects them. This separation is what makes conflict detection, resolution synthesis, and shadow-price attribution expressible — none of which are possible when "commit" and "state" are the same object.

The six-step decomposition above suggests where the separation bites: steps 1–4 produce the [[Diff and Delta|patch]] (the control input), step 6 produces the state (the named-graph snapshot), and the relationship between them is mediated by the state transition function — not collapsed into a single hash.

---
← [[Key Insight]] · [[Flexo MMS]] · [[Model]] · [[Diff and Delta]] · [[Merge]] · [[Conflict Resolution Problem Statement]]
