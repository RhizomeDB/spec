# The Postmodern Database (PomoDB) v0.1.0

## Editors

- [Quinn Wilton], [Fission Codes]
- [Brooklyn Zelenka], [Fission Codes]

## Authors

- [Quinn Wilton], [Fission Codes]
- [Brooklyn Zelenka], [Fission Codes]

## Dependencies

- [IPLD]
- [Pomo Logic]
- [Pomo RA]
- [WebNative File System]

## Appendices

- [Research]

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119].

# Abstract

PomoDB is a [content-addressable] database. It enables far-edge deployments (such as IoT and consumer devices) by synchronize heterogenous sets of end-to-end encrypted data between peers in an eventually consistent manner.

# 1. Introduction

## 1.1 Motivation

While existing databases increasingly treat network partitions as unavoidable disturbances to uptime, many mitigate these situations under a model that either assumes they will be bounded in length, or by allowing them to have some or all nodes cease operation. Such techniques are unsuitable for far-edge and local-first applications, where disorder is the norm, network partitions are ubiquitous and unbounded, and there's an uncompromising need for availability.

Modern environments increasingly involve unstable dynamic network topologies of heterogenous peers for which common definitions of consistency often do not apply. In this context, continued operation is not just desirable, but enforcing continuous consistency between all peers is a meaningless proposition. This begs the question: what if there was [no concept of "primary site"][A Certain Tendency of the Database Community] and all data were considered subjective relative to the viewer.

## 1.2 Approach

PomoDB addresses the above constraints with query engine semantics that guarantees eventual consistency across peers with access to the relevant data, but never requires nor assumes full synchronization between clients. This enables PomoDB to act as a sound foundation for globally distributed data with an indeterminate number of transient peers, with varied access patterns, and ever changing access to data.

## 1.3 Environments

PomoDB is designed to be:

- Run anywhere
- Transport agnostic
- Operate when disconnected from the network

Of particular interest are operating in browsers, mobile applications, and IoT devices.

# 2. Design

## 2.1 Relation

A relation is a set of tuples, where each component of the tuple is called an attribute, and can be referenced by an attribute name.

An n-ary relation contains n-tuples, which can each be written:

$\langle a_1: v_1, a_2: v_2, \dots , a_n: v_n \rangle$

Where $a_1, a_2, \dots , a_n$ give the names of each attribute, and $v_1, v_2, \dots , v_n$ are their values.

## 2.2 Time

PomoDB is a temporal database which internally models the progression of time as data changes, with each batch of data relating to a timestamp, called an epoch.

While epochs MUST be represented as an unsigned integer, runtimes MAY refine this representation in order to capture additional metadata about the progression of time.

All such representations MUST form a partial order, such that subsequent epochs are represented using subsequent timestamps. For example, runtimes MAY represent timestamps using a pair that tracks the iteration count of looping queries in the second component, while comparing these pairs using product order.

The database state is modeled as a function of time and the program under evaluation, and all tuples are associated with the timestamp at which they were computed.

PomoDB timestamps form a logical clock and hold no meaning to any other instances of PomoDB.

## 2.3 Content Addressing

As PomoDB is intended for use in distributed and decentralized deployments, it is important ensure the use of collision resistant identifiers when referring to tuples. For this purpose, a content addressing scheme is leveraged, and tuples are associated with a CID computed from their structure. The details behind this computation are available in [serialization].

The choice of CIDs here, rather than more common choices, like auto incrementing IDs or UUIDs, reflects PomoDB's goals in targeting distributed and decentralized environments, where coordination around the allocation of IDs can't be guaranteed, and where resilience against malicious and byzantine actors is required.

Since content addressing schemes are backed by cryptographically secure hash functions, their use here prevents forgery of IDs by attackers, and guarantees that CID-based dependencies between tuples will be acyclic.

These properties are further leveraged in the design and use of byzantine-fault tolerant CRDTs, as described in [CRDTs].

## 2.4 Query Engine

PomoDB has no specified query language. Instead, an intermediate representation based on the relational algebra, named [PomoRA], is defined.

Implementations MAY define their own user-facing query language, but they are RECOMMENDED to treat PomoRA as a common compilation target for all such languages.

An OPTIONAL Datalog variant, named [PomoLogic], is also described, along with an OPTIONAL runtime for PomoRA, named [PomoFlow].

## 2.5 Evaluation

Evaluation of PomoDB queries proceeds in timesteps, called epochs, which each compute a least fixed point over a batch of changes to the database.

The details behind this computation are runtime specific, however all runtimes MUST provide the following additional guarantees.

Each epoch is denoted by the timestamp succeeding the last, and begins by scheduling the program's [sources] for evaluation. These sources MAY run in any order, however any operations over their resulting contents MUST be deferred until their completion.

Upon computing a relation's fixed point, any [sinks] over that relation SHOULD be scheduled to run over the relation's contents, and evaluation of those sinks MUST be completed before evaluating the next epoch.

PomoDB queries MAY be implemented over incremental computations, in which case each epoch is RECOMMENDED to operate over deltas of the database, wherever possible. [PomoFlow] is an OPTIONAL runtime with such capabilities.

## 2.6 Sources

Sources act as ingress points for a PomoDB query, and introduce tuples from the outside world, such as by loading them from a local persistence layer, or by querying them from a remote data source such as [IPFS].

Sources can be queried as if they were [relation]s.

Implementations MAY define their own sources, but sources SHOULD be non-blocking, and are RECOMMENDED to perform any blocking or IO-intensive operations asynchronously.

Sources MAY emit deltas of tuples, if a runtime able to take advantage of incremental computation is being used, like [PomoFlow].

Implementations MAY also support user defined sources, such as to facilitate the integration of PomoDB into external systems for persistence or communication.

## 2.7 Sinks

Sinks act as egress points for a PomoDB query, and emit tuples to the outside world for further processing or storage.

Sinks can be inserted into as if they were [relation]s.

Implementations MAY define their own sinks, but sinks SHOULD be non-blocking, and are RECOMMENDED to perform any blocking or IO-intensive operations asynchronously.

Implementations MAY also support user defined sinks, such as to facilitate the integration of PomoDB into external systems for persistence or communication.

# 3 Components

PomoDB is composed of several cleanly separated layers. There is a distinction between the hard technical dependencies between these components and the way that this spec has decided to compose them for various pragmatic reasons.

## 3.1 Components 

The raw dependencies between components in the systems stack as follows:

``` 
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                         PomoDB                          │
│                           SDK                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│                 │ │                 │ │                 │
│   Query DSL 1   │ │   Query DSL 2   │ │   Query DSL 3   │
│                 │ │                 │ │                 │
└─────────────────┘ └─────────────────┘ └─────────────────┘
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                        Compiler                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│                 │ │                 │ │                 │
│     Datalog     │ │    Relational   │ │  Differential   │
│                 │ │     Algebra     │ │    Dataflow     │
│                 │ │                 │ │                 │
└─────────────────┘ └─────────────────┘ └─────────────────┘
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                        Pomo Store                       │
│                        EDB & IDB                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                Abstract Object Store                    │
│                Content Addressed Data                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

PomoDB choses to implement this as a the following stack:

``` 
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                         PomoDB                          │
│                           SDK                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│                 │ │                 │ │                 │
│   Query DSL 1   │ │   Query DSL 2   │ │   Query DSL 3   │
│                 │ │                 │ │                 │
└─────────────────┘ └─────────────────┘ └─────────────────┘
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                       Pomo Logic                        │
│                       Datalog IR                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                        Pomo RA                          │
│                     Query Planner                       │
│                                                         │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                        Pomo Flow                        │
│              Differential Dataflow Engine               │
│                                                         │
└─────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────┐
│                                                         │
│                        Pomo Store                       │
│                Content Addressed Tuples                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
┌───────────────────────────┐ ┌───────────────────────────┐
│                           │ │                           │
│   WebNative File System   │ │      In-Memory Store      │
│        Synced Store       │ │       Ephemeral Data      │
│                           │ │                           │
└───────────────────────────┘ └───────────────────────────┘
┌─────────────────┐  ┌────────────────────────────────────┐
│                 │  │                                    │
│  Remote Store   │  │             Local Store            │
│                 │  │                                    │
└─────────────────┘  └────────────────────────────────────┘
```

### 3.1.1 Pomo Logic

PomoDB is query language agnostic, and implementations MAY define arbitrary query frontends. [PomoLogic] is a variant of Datalog, designed for recursive query processing.

### 3.1.2 Pomo Flow

PomoFlow is a dataflow runtime for PomoDB. This design is especially suited for incrementalizing programs to efficiently compute over deltas to the input EDB.

### 3.1.3 Pomo RA

Relational algebra is the theory underpinning relational databases and the query languages against them. It provides a small core language of relational operators that serve as simple building blocks for more complex forms of query processing.

This specification describes PomoRA, a representation of the relational algebra intended for use as an intermediate representation for PomoDB query languages.

# 4 Extensional Database

The data layout stored to disk is the "extensional database" (EDB). This is the "ground truth" database, from which all other knowledge is derived via queries.

## 4.1 Types

### 4.1.1 Fact

The extensional database MUST only store facts as quads (4-tuples) in EAVC format. All fields are REQUIRED, though the causal set MAY be empty.

| Field         | Type          | Description                                        |
|---------------|---------------|----------------------------------------------------|
| **E**ntity ID | [`EntityID`]  | Entity ID                                          |
| **A**ttribute | [`Attribute`] | The name of the value's relationship to the entity |
| **V**alue     | [`Value`]     | The (primitive) value being associated             |
| **C**ausal    | `Set CID`     | Any causal links                                   |

``` haskell
data Fact = Fact
  { eid    :: Bytes
  , attr   :: Attribute
  , val    :: Value
  , causal :: Set CID
  }
```

The first three fields (entity, attribute, and value) are analogous to a subject-predicate-object statement. For example, "the sky is blue" MAY be represented as $\langle \textsf{skyEID}, \textsf{colour}, \textsf{blue} \rangle$.

#### 4.1.1.1 Implicit CID

Each tuple within a fact also has an implied [CID][content addressing]. This behaves as an index on all facts. Being derived from the hash of the fact means that the CID can always be rederived.

``` haskell
type CidIndex = Map CID Fact
```

As described in the section on [time], causal relationships are one way of representing order. This is the RECOMMENDED ordering mechanism since including hashes a priori implies a happened-after relatiship (assuming no known hash cycles).

Using the "sky is blue" example above, how woud that update for the evening?

$$
\begin{align}
\textsf{bafy...noon} &= \langle \textsf{skyEID}, \textsf{colour}, \textsf{blue}, \emptyset \rangle\\
\textsf{bafy...sunset} &= \langle \textsf{skyEID}, \textsf{colour}, \textsf{orange}, \{ \textsf{bafy...noon} \} \rangle\\
\textsf{bafy...night} &= \langle \textsf{skyEID}, \textsf{colour}, \textsf{black}, \{ \textsf{bafy...sunset} \} \rangle
\end{align}
$$

``` mermaid
flowchart RL
    sunset -- after --> noon
    midnight -- after --> sunset
```

### 4.1.2 Entity ID

An "entity" is some subject in the database that can have an attribute. 

``` haskell
newtype EntityID = EntityID Binary
```

### 4.1.3 Attribute

``` haskell
data Attribute
  = AttrInt   Integer -- e.g. Normal indices
  | AttrFloat Double  -- e.g. Fractional indices
  | AttrBin   Bytes
  | AttrText  UTF8
```

### 4.1.4 Value

The EDB supports the following primitive value types:

``` haskell
data Value
  = EntityID Bytes
  | Attr     Attribute
  | Bool     Boolean
  | Bin      Bytes
  | Int      Integer
  | Float    Double
  | Text     UTF8
  | Link     CID
```

<!-- Links -->

[A Certain Tendency of the Database Community]: https://arxiv.org/pdf/1510.08473.pdf
[Brooklyn Zelenka]: https://github.com/expede
[CRDTs]: pomo_db/CRDTs.md
[Fission Codes]: https://fission.codes
[IPFS]: https://ipfs.io
[IPLD]: https://ipld.io/specs/
[Pomo Logic]: https://github.com/RhizomeDB/PomoLogic
[Pomo RA]: https://github.com/RhizomeDB/PomoRA
[Quinn Wilton]: https://github.com/QuinnWilton
[RFC 2119]: https://datatracker.ietf.org/doc/html/rfc2119
[Research]: https://github.com/RhizomeDB/research
[Wasm Numbers]: https://webassembly.github.io/spec/core/syntax/types.html#syntax-numtype
[Wasm Opaque Reference Types]: https://webassembly.github.io/spec/core/syntax/types.html#reference-types
[Wasm Primitive Types]: https://webassembly.github.io/spec/core/appendix/index-types.html
[WebAssembly]: https://webassembly.org/
[WebNative File System]: https://github.com/wnfs-wg/spec
[`Attribute`]: #413-attribute
[`EntityID`]: #412-entity-id
[`Value`]: #414-value
[content addressing]: #24-content-addressing
[relation]: #22-relation
[serialization]: ./pomo_db/serialization.md
[sinks]: #28-sinks
[sources]: #27-sources
[time]: #22-time
