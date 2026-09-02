+++
date = '2026-08-25T10:00:00+05:30'
draft = false
title = 'SPDX looks Flat. But can it really represent a Hierarchical SBOM?'
categories = ['Tools', 'Best Practices', 'Standards']
tags = ['SBOM', 'sbomasm', 'SPDX', 'CycloneDX', 'Merge Strategies', 'Hierarchical Merge','SBOM Standards']
author = 'Vivek Sahu'
description = 'Exploring how SPDX relationship types `CONTAINS` and `DEPENDS_ON` can represent hierarchical SBOM semantics, and what we learned from the SPDX Implementors meeting about multi-format merge strategy parity.'
slug = 'spdx-looks-flat-but-can-it-really-represent-a-hierarchical-sbom'
+++

Hey SBOM enthusiasts 👋,

If you've been working with SBOMs for a while, you already know that merging multiple SBOMs isn't just about dumping all the components into one file.

The real challenge is preserving the **semantics meaning of the merge strategies**.

Are they dependencies?
Are they sub-components of another component?
Are they independent products being assembled together?
Or are we simply trying to flatten everything into one unified component list?

This is exactly why sbomasm supports multiple merge strategies. We previously discussed how [Hierarchical, Flat, Assembly, and Augment Merge work](/posts/understanding-sbomasm-merge-strategies/) and when each strategy makes sense. In particular, Hierarchical Merge allows each input SBOM to retain its own component hierarchy under a new root while preserving its dependency relationships.

But while implementing these strategies across SBOM formats, we ran into an interesting question:

> Can we represent the same merge semantics in SPDX that we already represented for CycloneDX?

At first glance, SPDX makes this look difficult. And that's where things get interesting.

## The Problem: CycloneDX has nesting. SPDX looks flat

Let's start with a simple example.

Imagine we have two SBOMs:

```text
SBOM A
└── Frontend
    ├── React
    ├── Axios
    └── Nginx

SBOM B
└── Backend
    ├── Express
    ├── Mongoose
    └── JWT
```

With a Hierarchical Merge, we want to produce something like:

```text
My Application
├── Frontend
│   ├── React
│   ├── Axios
│   └── Nginx
└── Backend
    ├── Express
    ├── Mongoose
    └── JWT
```

This is straightforward in CycloneDX because CycloneDX has an actual component hierarchy structure. Components can contain sub-components, so the structure itself tells us what belongs where. **But SPDX takes a different approach.**

In SPDX structurally, packages/elements are represented in a flat collection, and **relationships describe how those elements are connected**. SPDX defines relationship types such as `DEPENDS_ON`, `CONTAINS` ad many more to express different kinds of connections.

So the SPDX packages representation looks more like flatten in structure:

```text
Frontend
React
Axios
Nginx
Backend
Express
Mongoose
JWT
```

There is no nested `components[]` unlike CycloneDX tree telling us that React belongs underneath Frontend. Instead in SPDX, we need to derive that information from relationships. And that led us to an important question.

## First Question: Can `CONTAINS` represent nesting?

The first question was about hierarchy itself. If SPDX doesn't support nesting structure, can we use:

```text
CONTAINS
```

relationshipType to express that one element is a sub-component of another element?

> The answer from the SPDX Implementors Meeting discussion was **yes**.

The SPDX relationship model provides **CONTAINS** specifically for expressing containment relationships. In other words, while the SPDX document remains structurally flat, the relationship graph can express that one element contains another.

So the below CycloneDX hierachy of sub-component:

```text
Frontend
└── React
```

can equivalently represent the same semantic relationship in SPDX as:

```text
Frontend ──CONTAINS──> React
```

And:

```text
Backend ──CONTAINS──> Express
```

This gives us the first piece of the puzzle. But we still had another question.

## The more interesting Question: Can two Relationships co-exist?

Now consider a real software dependency graph. A component can have more than one semantic relationship with another component.

**For example**, suppose React is a component contained inside Frontend, and Frontend also depends on React. We may want to express both facts:

```text
Frontend ──CONTAINS──> React
Frontend ──DEPENDS_ON──> React
```

These relationships are not saying the same thing.

- **CONTAINS** tells us about structure: React is part of the Frontend component hierarchy.

- **DEPENDS_ON** tells us about dependency semantics: Frontend depends on React.

So the question we raised with the SPDX Implementors community was whether SPDX allows these two relationship types to co-exist between the same source and target.

> The conclusion from the Implementors discussion was **yes**.

And that was the second piece of the puzzle.

## Now let's connect the dots...

Once these two questions were resolved, the problem started looking very different. We originally thought:

> SPDX is flat, so hierarchical merging may not be possible in the same way as CycloneDX.

But now we can look at it differently. **SPDX may be structurally flat**, but its **relationship model is rich** enough to express the connections b/w different elements.

**For example**:

```text
                 My Application
                       |
                  DEPENDS_ON
                 /         \
                /           \
          Frontend         Backend
             |                |
          CONTAINS          CONTAINS
             |                |
           React           Express
             |                |
          DEPENDS_ON       DEPENDS_ON
```

The physical document is still flat. But the semantic model isn't. The hierarchy or strucutre is represented using **CONTAINS**. The dependency graph is represented using **DEPENDS_ON**. And both relationships can co-exist when they describe different semantics between the same elements.

That is the key insight.

## So what does this mean for Hierarchical Merge?

Let's go back to our original example.

We have:

```text
SBOM A
Frontend
├── React
├── Axios
└── Nginx
```

and:

```text
SBOM B
Backend
├── Express
├── Mongoose
└── JWT
```

A Hierarchical Merge should **preserve the individual SBOM hierarchies while also preserving their dependency relationships**.

- In CycloneDX, we can represent this directly using **nested components**.
- In SPDX, we can represent the same semantics through **relationships** of type `CONTAINS`.

**Conceptually**:

```text
Root
 |
 +── DEPENDS_ON ──> Frontend
 |                    |
 |                    +── CONTAINS ──> React
 |                    +── CONTAINS ──> Axios
 |                    +── CONTAINS ──> Nginx
 |
 +── DEPENDS_ON ──> Backend
                      |
                      +── CONTAINS ──> Express
                      +── CONTAINS ──> Mongoose
                      +── CONTAINS ──> JWT
```

And the original dependency relationships can continue to exist independently.

**For example**:

```text
Frontend ──DEPENDS_ON──> React
Frontend ──DEPENDS_ON──> Axios
Backend  ──DEPENDS_ON──> Express
```

Now we have both pieces of information:

- Where does a component belong? -> **CONTAINS**
- What does a component depends on? -> **DEPENDS_ON**

This is exactly what we were trying to preserve with Hierarchical Merge.

## What about the other merge strategies?

Once we understand this relationship-based model, the other merge strategies become easier to reason about too.

### Assembly Merge

Assembly Merge has a slightly different goal.

Here, the primary components of input SBOMs become sub-components of the new root(i.e. new primary component), while the rest of the component information remains available at the appropriate level.

In SPDX, that means the synthetic root can use:

```text
Root
├── CONTAINS ──> Product A
└── CONTAINS ──> Product B
```

This matches the semantic idea of an assembled collection rather than saying that the root itself depends on those products.

### Flat Merge

Flat Merge is different again. Here, we don't want to preserve the hierarchy. Everything is flattened. But we do want to preserve the dependency graph.

So the important relationships are:

```text
Root
├── DEPENDS_ON ──> Primary A
└── DEPENDS_ON ──> Primary B

Primary A
└── DEPENDS_ON ──> A1

A1
└── DEPENDS_ON ──> A2

Primary B
└── DEPENDS_ON ──> B1
```

So the three strategies aren't simply different ways of moving packages around. They are different **semantic views** of the same underlying SBOM information.

## The bigger lesson: Flat structure doesn't mean Flat semantics

This was probably the most interesting part of this investigation for me.

When we say:

> "SPDX is flat."

it is easy to interpret that as:

> "SPDX cannot represent hierarchy."

But those aren't the same thing.

SPDX doesn't need a nested JSON structure to represent a hierarchy. It can represent that hierarchy through its relationship graph.

Think about it this way.

CycloneDX gives us:

- > Structure + Relationships

while SPDX gives us:

- > Elements + Relationships

The hierarchy in SPDX is therefore **derived from relationships** rather than encoded directly in the physical structure.

And because different relationship types carry different semantics, we can represent multiple dimensions of information without changing the underlying flat structure. That is what makes this approach possible.

## What this means for sbomasm tool ?

This investigation gives us a clear direction for making the SPDX merge strategies consistent with their CycloneDX counterparts.

The goal isn't to make the generated SPDX document look like a CycloneDX document. That's impossible — and unnecessary. The goal is to make sure that:

> The same merge strategy carries the same semantic meaning regardless of the SBOM format.

For sbomasm, that means:

| Merge Strategy | CycloneDX | SPDX |
|--------------|-----------|------|
| Hierarchical | Nested components + dependencies | CONTAINS + DEPENDS_ON relationships |
| Assembly | Primary components nested under root | Root CONTAINS primary components |
| Flat | Flattened components + dependency graph | Flat elements + preserved DEPENDS_ON relationships |
| Augment | Enrich existing SBOM | Enrich existing SBOM |

The representation is different but semantics should not be. And that is the real objective behind the work.

## One more important distinction

There is one thing worth making absolutely clear.

We should not say:

> "SPDX supports native component nesting."

It doesn't work that way.

SPDX remains a **relationship-driven** model. The elements themselves are not physically nested in the document in the way CycloneDX components can be nested.

A more accurate statement is:

> SPDX does not have structural nesting like CycloneDX, but its relationship model can represent nesting semantics through CONTAINS relationship type.

That's a subtle distinction, but an important one. 

## Wrapping Up...

What started as a small inconsistency between SPDX and CycloneDX merge strategies turned into a much more interesting question about how SBOM formats represent relationships. CycloneDX makes hierarchy explicit through its component structure, whereas SPDX takes a different approach.

Its structure is flat, but its relationship model provides the vocabulary needed to describe how those elements relate to one another.

And once we understood that:

- **CONTAINS** → structural relationship
- **DEPENDS_ON** → dependency relationship

## Resources

- [sbomasm GitHub repository](https://github.com/interlynk-io/sbomasm)
- [sbomasm issue #344 — SPDX merge strategy inconsistencies](https://github.com/interlynk-io/sbomasm/issues/344)
- [Understanding SBOM Merge Strategies](/posts/understanding-sbomasm-merge-strategies/)
- [SPDX Specifications](https://spdx.dev/)
- [SPDX 2.2 Specification — Relationships](https://spdx.github.io/spdx-spec/v2.2.2/relationships-between-SPDX-elements/)
- [SPDX 3.0.1 Core Specification](https://spdx.github.io/spdx-spec/v3.0.1/)

If you loved this project, show the love back by starring ⭐ the [repo](https://github.com/interlynk-io/sbomasm).

Have questions or need help with a specific use case? File an [issue](https://github.com/interlynk-io/sbomasm/issues/new), we'd love to help!
