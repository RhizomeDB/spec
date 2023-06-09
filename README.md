# The Postmodern Database (PomoDB) v0.1.0

## Editors

- [Quinn Wilton], [Fission Codes]
- [Brooklyn Zelenka], [Fission Codes]

## Authors

- [Quinn Wilton], [Fission Codes]
- [Brooklyn Zelenka], [Fission Codes]

## Dependencies

- [IPLD]
- [PomoLogic]
- [PomoRA]
- [WebNative File System]

## Appendices

- [Research]

# Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119].

# Abstract

PomoDB is a [content-addressable] database. It enables far-edge deployments (such as IoT and consumer devices) by synchronize heterogenous sets of end-to-end encrypted data between peers in an eventually consistent manner.

# 1. Introduction

> A concept is a brick. It can be used to build a courthouse of reason. Or it can be thrown through the window.
>
> ― [Deleuze] & [Guattari], [A Thousand Plateaus: Capitalism and Schizophrenia] 

## 1.1 Motivation

While existing databases increasingly treat network partitions as unavoidable disturbances to uptime, many mitigate these situations under a model that either assumes they will be bounded in length, or by allowing them to have some or all nodes cease operation. Such techniques are unsuitable for far-edge and local-first applications, where disorder is the norm, network partitions are ubiquitous and unbounded, and there's an uncompromising need for availability.

Modern environments increasingly involve unstable dynamic network topologies of heterogenous peers for which common definitions of consistency often do not apply. In this context, continued operation is not just desirable, but enforcing continuous consistency between all peers is a meaningless proposition. This begs the question: what if there was [no concept of "primary site"][A Certain Tendency of the Database Community] and all data were considered subjective relative to the viewer.

## 1.2 Approach

> No single theory ever agrees with all the facts in its domain.
> 
> — [Paul Feyerabend], [Against Method]

PomoDB addresses the above constraints with query semantics that guarantee eventual consistency between all peers with access to the relevant data, but never requires nor assumes full synchronization between clients. This enables PomoDB to act as a sound foundation for globally distributed data with an indeterminate number of transient peers, with varied access patterns, and ever changing access to data.

Historically databases have assumed an objective worldview: there is a single, correct, _objective_ way to look at data. This can be convenient, but is also overly simplistic for many cases. As application state becomes increasingly [distributed][Fallacies of distributed computing], partition-tolerant, private, and remixable, modeling data as _subjective_ makes increasing sense. This is in line with the [postmodern worldview][Postmodernism], which recognizes that there is no one true way to look at things, but rather multiple interpretations that are useful for different purposes (including [mathematics][Pomo Math] and science). The only objective facts in PomoDB are the facts (as stements) themselves, and their claimed [causal ordering][B-Series].

This approach lets one treat access control, latency, and partitions as equivalent, and embrace (not just tollerate) all three. Conceptually there is [one, unchanging, universal database][Eternalism], and various observers merely come to learn of parts of the database (new data, new interpretations) over time. However, like in physics, this is not an instantaneous or evenly distributed process: some observers will learn of data before others, and may interact with asymmetric information. An observer doesn't need to learn of the entire universe to use the database: a subjective view is sufficient for the majority of cases. When stronger consistency  is desired, the relevant observers can synchrnize the relevant subset of context.

## 1.3 Environments

PomoDB is designed to be:

- Run anywhere
- Transport agnostic
- Operate when disconnected from the network

Of particular interest are operating in browsers, mobile applications, and IoT devices.

## 1.4 Classification

PomoDB can be seen as a maximally-sharded schemaless database, a meta-circular CRDT framework, or both simultaneously. It decouples the raw data from interpretation, and expresses [CRDTs as database queries][Keep Calm and CRDT On]. Other than being able to benefit from the last 50+ years of database research, this flexibility makes the database itself user extensible.

# 2. Design

## 2.1 Relation

A relation is a set of tuples, where each component of the tuple is called an attribute, and can be referenced by an attribute name.

An n-ary relation contains n-tuples, which can each be written: $\langle a_1: v_1, a_2: v_2, \dots , a_n: v_n \rangle$, where $a_1, a_2, \dots , a_n$ give the names of each attribute, and $v_1, v_2, \dots , v_n$ are their values.

## 2.2 Time

> Cause and effect are two sides of one fact.
> 
> Ralph Waldo Emerson 

PomoDB is a temporal database which internally models the progression of time as data changes, with each batch of data relating to a timestamp, called an epoch.

While epochs MUST be represented as an unsigned integer, runtimes MAY refine this representation in order to capture additional metadata about the progression of time.

All such representations MUST form a partial order, such that subsequent epochs are represented using subsequent timestamps. For example, runtimes MAY represent timestamps using a pair that tracks the iteration count of looping queries in the second component, while comparing these pairs using product order.

The database state is modeled as a function of time and the program under evaluation, and all tuples are associated with the timestamp at which they were computed.

PomoDB timestamps form a logical clock and hold no meaning to any other instances of PomoDB. The static nature of these links means that queries can reason from causes to effects, or from effects to their causes.

## 2.3 Content Addressing

As PomoDB is intended for use in distributed and decentralized deployments, it is important ensure the use of collision resistant identifiers when referring to tuples. For this purpose, content addressing scheme is used. Tuples are associated with a CID computed from their structure. The details behind this computation are available in [serialization].

The choice of CIDs as primary key — rather than more familiar choices such as auto incrementing IDs or UUIDs — reflects PomoDB's goals in targeting distributed and decentralized environments, where coordination around the allocation of IDs can't be guaranteed, and where resilience against malicious and byzantine actors is required.

Since content addressing schemes are backed by cryptographically secure hash functions, their use here prevents forgery of IDs by attackers, and guarantees that CID-based dependencies between tuples will always be acyclic.

These properties further enable [byzantine-fault tolerant CRDTs][BFT-CRDTs].

## 2.4 Query Engine

PomoDB has no specified high-level query language. An intermediate representation based on datalog is defined instead ([PomoLogic]). Implementations MAY define their own user-facing query language, but they are RECOMMENDED to treat [PomoLogic] as a common compilation target for all such languages.

An OPTIONAL relational runtime that MAY be compiled from [PomoLogic] is described in [PomoRA]. This can be further optimized with a dataflow runtime for incrementalization.

## 2.5 Evaluation

Evaluation of PomoDB queries proceeds in timesteps ("epochs") which compute a least fixed point over a batch of changes to the database.

The details behind this computation are runtime specific, however all runtimes MUST provide the following additional guarantees.

Each epoch is denoted by the timestamp succeeding the last, and begins by scheduling the program's [sources] for evaluation. These sources MAY run in any order, however any operations over their resulting contents MUST be deferred until their completion.

Upon computing a relation's fixed point, any [sinks] over that relation SHOULD be scheduled to run over the relation's contents, and evaluation of those sinks MUST be completed before evaluating the next epoch.

PomoDB queries MAY be implemented over incremental computations, in which case each epoch is RECOMMENDED to operate over deltas of the database, wherever possible. PomoFlow is an OPTIONAL runtime with such capabilities.

## 2.6 Sources

Sources act as ingress points for a PomoDB query, and introduce tuples from the outside world, such as by loading them from a local persistence layer, or by querying them from a remote data source such as [IPFS].

Sources can be queried as if they were [relation]s.

Implementations MAY define their own sources, but sources SHOULD be non-blocking, and are RECOMMENDED to perform any blocking or IO-intensive operations asynchronously.

Sources MAY emit deltas of tuples, if a runtime able to take advantage of incremental computation is being used, like PomoFlow

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

PomoDB chooses to implement this as a the following stack:

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

This specification describes [PomoRA], a representation of the relational algebra intended for use as an intermediate representation for PomoDB query languages.

# 4 Extensional Database

The data layout stored to disk is the "extensional database" (EDB). This is the "ground truth" database, from which all other knowledge is derived via queries.

## 4.1 Types

### 4.1.1 Primitive Types

The primitive types supported by PomoDB MUST be as below. Note that this is at the layer of PomoDB, and the concrete persisted encoding (e.g. JSON, CBOR, Protobuf) MAY be different.

| Name    | Size      | Descrition | 
|---------|-----------| -----------| 
| Boolean | One bit   | A boolean value.
| Integer | 64-bit    | Signed integer |
| Double  | 64-bit    | Double-precision floating-point number, as described in [IEEE 754-2019] |
| String  | Unbounded | [UTF8], including empty strings |
| Bytes   | Unbounded | Binary octets |
| CID     | Unbounded | Avoiding the identity hash is RECOMMENDED |

Note that there is no primitive `unit` or `null`. Emulating these constructs in data modeling is NOT RECOMMENDED, as they are semantically anemic and alternative modelings are nearly always available. For example, a common use case for using `null` is unset a field. CRDTs SHOULD use history-preserving tombstoning for this purpose instead.

### 4.1.2 Fact

The extensional database MUST only store facts as quads (4-tuples) in EAVC[^spoc] format. All fields are REQUIRED, though the causal set MAY be empty.

[^spoc]: This is [sometimes called SPOC][Hexastore] (🖖) for ["subject-predicate-object-context"][Named Graphs]

| Field         | Type          | Description                                        |
|---------------|---------------|----------------------------------------------------|
| **E**ntity ID | `Bytes`       | [Entity ID]                                        |
| **A**ttribute | [`Attribute`] | The name of the value's relationship to the entity |
| **V**alue     | [`Value`]     | The (primitive) value being associated             |
| **C**ausal    | `Set CID`     | Any causal links                                   |

``` typescript
type Fact = {
  eid: Bytes;
  attr: Attribute;
  val: Value;
  causal: Set<Cid>;
}
```

The first three fields (entity, attribute, and value) are analogous to a subject-predicate-object statement. For example, "the sky is blue" MAY be represented as $\langle \textsf{skyEID}, \textsf{color}, \textsf{blue} \rangle$.

#### 4.1.2.1 Implicit CID

Each tuple within a fact also has an implied [CID][content addressing]. This behaves as an index on all facts. Being derived from the hash of the fact means that the CID can always be rederived.

``` typescript
type CidIndex = {[Cid]: Fact}
```

As described in the section on [time], causal relationships are one way of representing order. This is the RECOMMENDED ordering mechanism since including hashes a priori implies a happened-after relationship (assuming no known hash cycles).

Using the "sky is blue" example above, how would that be updated for the evening?

$$
\begin{align}
  \textsf{bafy...noon} &= \langle \textsf{skyEID}, \textsf{color}, \textsf{blue}, \emptyset \rangle\\
  \textsf{bafy...sunset} &= \langle \textsf{skyEID}, \textsf{color}, \textsf{orange}, \{ \textsf{bafy...noon} \} \rangle\\
  \textsf{bafy...night} &= \langle \textsf{skyEID}, \textsf{color}, \textsf{black}, \{ \textsf{bafy...sunset} \} \rangle
\end{align}
$$

``` mermaid
flowchart RL
    sunset   -- after --> noon
    midnight -- after --> sunset
```

#### 4.1.2.2 Example Graph

| CID      | Entity ID | Attribute  | Value      | Cause      |
|----------|-----------|------------|------------|------------|
| bafy-abc | 789       | home       | Entity 456 | []         |
| bafy-def | 246       | home       | Entity 357 | []         |
| bafy-jkl | 456       | is         | city       | []         |
| bafy-mno | 456       | name       | Vancouver  | []         |
| bafy-pqr | 357       | name       | Calgary    | []         |
| bafy-stu | 246       | last_name  | Zelenka    | []         |
| bafy-vwx | 357       | is         | city       | []         |
| bafy-yzA | 789       | first_name | Boris      | []         |
| bafy-BCD | 246       | first_name | Brooklyn   | []         |
| bafy-EFG | 246       | home       | Entity 456 | [bafy-def] |
| bafy-GHI | 789       | last_name  | Mann       | []         |

``` mermaid
flowchart TB
    subgraph Legend
      direction TB

      entity{{Entity}} --- att([Attribute]):::attr ----> Value
      
      %% Layout hack
      att ~~~ cause ~~~ att2
      entity ~~~ cause

      att2 -.- cause>after/cause]:::attr -.-> att
      att2  --> AnotherValue
      entity --- att2([AnotherAttribute]):::attr
    end

    Legend ~~~ entityBZ

    entityBZ{{246}} --- bzFN([first name]):::attr --> Brooklyn
    entityBZ --- bzLN([last name]):::attr --> Zelenka

    entityBZ --- bzOldCity([home]):::attr --> entityCgy
    entityBZ --- bzNewCity([home]):::attr --> entityVC

    bzNewCity -.- after>after/cause]:::attr .-> bzOldCity

    entityBM --- bmCity([home]):::attr --> entityVC
    entityBM{{789}} --- bmFN([first name]):::attr --> Boris
    entityBM --- bmLN([last name]):::attr --> Mann

    entityCgy --- calName([name]):::attr --> Calgary
    entityCgy{{357}} --- calIs([is]):::attr --> city

    entityVC{{456}} --- vcIs([is]):::attr --> city
    entityVC --- vcName([name]):::attr --> Vancouver

    style entityBM fill:orange
    style entityBZ fill:purple
    style entityVC fill:blue
    style entityCgy fill:red

    classDef attr fill:lightgrey,color:grey,stroke:none;
```

### 4.1.3 Entity ID

An "entity" is some subject in the database that can have an attribute. Since names are not unique (and are in fact an attribute), each entity needs a unique identity. Using a random number of at least 128-bits when generating a fresh entity is is RECOMMENDED.

``` typescript
type EntityID = Bytes
```

### 4.1.4 Attribute

Attributes MUST be an integer, double-precision float, UTF8, or binary.

``` typescript
type Attribute
  = Integer // e.g. Normal indices
  | Double  // e.g. Fractional indices
  | Utf8
  | Bytes
```

### 4.1.5 Value

The EDB supports the following primitive value types:

``` typescript
type Value
  = Boolean
  | Integer
  | Double
  | Utf8
  | Cid
  | Bytes
```

# 5 Prior Art

This is a large amount of prior art in this area. Below are a few resources that were either direct influences, or frequently brought up as comparisons to PomoDB. They are presented here alphabetically:

## 5.1 [Automerge]

## 5.2 [Berkley Orders of Magnitude][BOOM]

## 5.3 [Datasette]

## 5.4 [Datomic] & [Datahike]

## 5.5 [Mentat]

## 5.6 [Project Cambria]

## 5.7 [RDF]

## 5.8 [Soufflé]

<!-- Links -->

[A Certain Tendency of the Database Community]: https://arxiv.org/pdf/1510.08473.pdf
[A Thousand Plateaus: Capitalism and Schizophrenia]: https://en.wikipedia.org/wiki/A_Thousand_Plateaus
[AMBROSIA]: https://www.microsoft.com/en-us/research/uploads/prod/2018/12/AmbrosiaPaper.pdf
[Against Method]: https://en.wikipedia.org/wiki/Against_Method
[Automerge]: https://automerge.org/
[B-Series]: https://en.wikipedia.org/wiki/B-theory_of_time 
[BFT-CRDTs]: https://martin.kleppmann.com/papers/bft-crdt-papoc22.pdf
[BOOM]: http://boom.cs.berkeley.edu/
[Brooklyn Zelenka]: https://github.com/expede
[Datahike]: https://github.com/replikativ/datahike
[Datasette]: https://datasette.io/
[Datomic]: https://www.datomic.com/
[Deleuze]: https://en.wikipedia.org/wiki/Gilles_Deleuze
[Eternalism]: https://en.wikipedia.org/wiki/Eternalism_(philosophy_of_time)
[Fallacies of distributed computing]: https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing
[Fission Codes]: https://fission.codes
[Guattari]: https://en.wikipedia.org/wiki/F%C3%A9lix_Guattari
[Hexastore]: https://people.csail.mit.edu/tdanford/6830papers/weiss-hexastore.pdf
[IPFS]: https://ipfs.io
[IPLD]: https://ipld.io/specs/
[Keep Calm and CRDT On]: https://www.vldb.org/pvldb/vol16/p856-power.pdf
[Laurent Binet]: https://en.wikipedia.org/wiki/Laurent_Binet
[Mentat]: https://mozilla.github.io/mentat/about
[Named Graphs]: https://en.wikipedia.org/wiki/Named_graph#Named_graphs_and_quads
[Paul Feyerabend]: https://en.wikipedia.org/wiki/Paul_Feyerabend
[Pomo Math]: https://link.springer.com/chapter/10.1007/978-3-319-12688-3_68
[PomoLogic]: https://github.com/RhizomeDB/PomoLogic
[PomoRA]: https://github.com/RhizomeDB/PomoRA
[Postmodernism]: https://en.wikipedia.org/wiki/Postmodernism
[Project Cambria]: https://www.inkandswitch.com/cambria/
[Quinn Wilton]: https://github.com/QuinnWilton
[RDF]: https://en.wikipedia.org/wiki/Resource_Description_Framework
[RFC 2119]: https://datatracker.ietf.org/doc/html/rfc2119
[Research]: https://github.com/RhizomeDB/research
[Soufflé]: https://souffle-lang.github.io/
[The Seventh Function of Language]: https://fr.wikipedia.org/wiki/La_Septi%C3%A8me_Fonction_du_langage
[UTF8]: https://www.unicode.org/versions/Unicode15.0.0/
[Wasm Numbers]: https://webassembly.github.io/spec/core/syntax/types.html#syntax-numtype
[Wasm Opaque Reference Types]: https://webassembly.github.io/spec/core/syntax/types.html#reference-types
[Wasm Primitive Types]: https://webassembly.github.io/spec/core/appendix/index-types.html
[WebAssembly]: https://webassembly.org/
[WebNative File System]: https://github.com/wnfs-wg/spec
[`Attribute`]: #413-attribute
[`Entity ID`]: #412-entity-id
[`Value`]: #414-value
[content addressing]: #24-content-addressing
[relation]: #22-relation
[serialization]: ./pomo_db/serialization.md
[sinks]: #28-sinks
[sources]: #27-sources
[time]: #22-time
