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

Historically databases have assumed an objective worldview: there is a single, correct, _objective_ way to look at data. This can be convenient, but is also overly simplistic for many cases. As application state becomes increasingly [distributed][Fallacies of distributed computing], partition-tolerant, private, and remixable, modeling data as _subjective_ makes increasing sense. This is in line with the [postmodern worldview][Postmodernism], which recognizes that there is no one true way to look at things, but rather multiple interpretations that are useful for different purposes (including [mathematics][Pomo Math] and science). The only objective facts in PomoDB are the quads (as statements) themselves, and their claimed [causal ordering][B-Series].

This approach lets one treat access control, latency, and partitions as equivalent, and embrace (not just tolerate) all three. Conceptually there is [one, unchanging, universal database][Eternalism], and various observers merely come to learn of parts of the database (new data, new interpretations) over time. However, like in physics, this is not an instantaneous or evenly distributed process: some observers will learn of data before others, and may interact with asymmetric information. An observer doesn't need to learn of the entire universe to use the database: a subjective view is sufficient for the majority of cases. When stronger consistency  is desired, the relevant observers can synchronize the relevant subset of context.

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

| Name    | Size      | Description                                                                                                     | 
|---------|-----------| ----------------------------------------------------------------------------------------------------------------| 
| Boolean | One bit   | A boolean value.                                                                                                |
| Integer | 64-bit    | A signed integer.                                                                                               |
| Double  | 64-bit    | [Double-precision] floating-point number, as described in [IEEE 754-2019]. `NaN`s SHOULD NOT be considered valid. |
| String  | Unbounded | A [UTF8] string. Empty strings MUST be considered valid.                                                        |
| Bytes   | Unbounded | Zero or more binary octets.                                                                                     |
| CID     | Unbounded | A [CID]. Avoiding the identity hash is RECOMMENDED.                                                             |

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

An "entity" MUST represent some subject in the database that can have an attribute. Since names are not unique (and are in fact an attribute), each entity needs a unique identity. Using a random number of at least 128-bits when generating a fresh entity is is RECOMMENDED.

``` typescript
type EntityID = Bytes
```

Especially since PomoDB allows for creation IDs without coordination, a real-world entity MAY be represented by multiple entity IDs. This case SHOULD be handed at the query layer by selecting for all relevant entity IDs.

### 4.1.4 Attribute

Attributes MUST be an integer, double-precision float, UTF8 string, or binary:

``` typescript
type Attribute
  = Integer // e.g. Normal indices
  | Double  // e.g. Fractional indices
  | Utf8
  | Bytes
```

### 4.1.5 Value

The EDB MUST support the following primitive value types:

``` typescript
type Value
  = Boolean
  | Integer
  | Double
  | Utf8
  | Cid
  | Bytes
```

### 4.1.6 Cause

Causal links MUST be expressed as an unordered set of CIDs, where those CIDs are other PomoDB fact CIDs. CIDs MUST be deduplicated at read time (though deduplicating and sorting at write time is RECOMMENDED).

``` typescript
type Cause = Set<Cid>
```

## 4.2 Graph Topologies

All relations in PomoDB MAY be expressed in several ways, but common terminology around graphs is RECOMMENDED as it is very clear and pictorial.

### 4.2.1 Grouping: Set & Stars

Grouping by entity ID, attribute, or value all produce sets. In graph terms, this is expressed as a "star" or "hub and spoke" topology.

| Role  | Description                                              |
|-------|----------------------------------------------------------|
| Hub   | The center of the relation / the item being "grouped by" |
| Spoke | The thing being related                                  |

``` mermaid
flowchart BT
    sun(("☀️"))

    earth(("🌏"))
    saturn(("🪐"))
    alien(("👽"))
    uap(("🛸"))
    meteor(("☄️"))

    earth --- sun
    sun --- saturn
    alien --- sun
    sun --- uap
    meteor --- sun
```

Multiple related hubs are possible.

``` mermaid
flowchart BT
    sun(("☀️"))

    earth(("🌏"))
    saturn(("🪐"))
    alien(("👽"))
    uap(("🛸"))
    meteor(("☄️"))

    earth ------ sun
    sun --- saturn
    alien --- sun
    sun --- uap
    meteor --- sun

    moon(("🌖")) --- earth
    satelite(("🛰️")) --- earth
    earth --- astro(("👩‍🚀"))
```

### 4.2.2 Ordering & Causality

An important order (or "sort") in PomoDB is causal order. This builds up a graph of pointers in the [`Cause`] field.

Below is a example causal graph from three writers. The grouping in this diagram is not important, but is presented this way here for explanatory purposes. In reality, if two writers commit the same fact to the database with the same causal history, they would be be deduplicated since they would have the same CID.

``` mermaid
flowchart RL
    classDef genesis fill: blue;
    classDef head    fill: green;

    subgraph Alice
        almond[bafy...almond]:::genesis
        apple[bafy...apple]
        adobo[bafy...adobo]
        agave[bafy...agave]
        ambrosia[bafy...ambrosia]
        avocado[bafy...avocado]
        asiago[bafy...asiago]
    end

    subgraph Bob
        bacon[bafy...bacon]:::genesis
        bagel[bafy...bagel]
        banana[bafy..banana]
        bean[bafy...bean]
        bun[bafy...bun]
        brie[bafy...brie]
        butter[bafy...butter]
        brine[bafy...brine]
        baklava[bafy...baklava]
        berry[bafy...berry]:::head
    end

    subgraph Carol
      cake[bafy...cake]
      cinnamon[bafy...cinnamon]
      cherry[bafy...cherry]
      calamari[bafy...calamari]
      carrot[bafy...carrot]
      chocolate[bafy...chocolate]
      coffee[bafy...coffee]:::head
    end

    apple --> almond
    adobo --> apple
    agave --> bagel
    agave --> adobo
    bagel --> bacon
    banana --> bagel
    avocado ----> ambrosia
    cake --> banana
    cinnamon ==> cake
    cherry --> cinnamon
    calamari --> cherry
    carrot ===> calamari
    chocolate ---> carrot
    chocolate ==> avocado
    calamari ==> bean
    berry --> brie
    bun ==> carrot
    butter --> bun
    brine --> butter
    baklava --> brine
    brie --> baklava
    brie --> asiago
    asiago ----> avocado
    bean -----> banana
    ambrosia ==> cinnamon
    cake ==> agave
    avocado --> calamari
    coffee -----> chocolate
    baklava ==> chocolate

    %% Transative Path Styles
        %% baklava -> chocolate -> avodcado
           linkStyle 13 stroke: DodgerBlue;
           linkStyle 28 stroke: DodgerBlue;

        %% calamari -> bean -> adobo
            linkStyle 11 stroke: orange;
            linkStyle 14 stroke: orange;
            linkStyle 16 stroke: orange;

        %% ambrosia -> cinnamon -> cake -> agave
            linkStyle 8  stroke: deeppink;
            linkStyle 24 stroke: deeppink;
            linkStyle 25 stroke: deeppink;
```

Note the direction of the arrows: they point from an event to their antecedent cause (i.e. they are pointers like in an EVAC quad). This is sometimes confusing, since we normally reason from cause to effect, but all facts in PomoDB are by their nature "in the causal past". Items in a node's causal history are called its "ancestors". Nodes that depend on a fact are called its "descendants". Graphs MAY be rooted.

#### 4.2.2.1 Genesis Nodes

The "earliest" facts in the graph above are `bafy...almond` and `bafy...bacon` (highlighted in blue), as they have no asserted causes. These are called "genesis nodes" (or simply "geneses"). In formal terms, these are "causal sinks". These nodes MAY be treated as objectively the earliest in the graph: since as causal information is asserted at write-time, these cannot be earlier, unlisted facts added or discovered later.

There MUST be at least one genesis node in an inhabited graph (even if it is a single-node graph). There MAY be an unbounded number of concurrent genesis nodes.

#### 4.2.2.2 Head Nodes

The "latest" facts in the above graph are `bafy...berry`, and `bafy...coffee` (highlighted in green). These are called "head nodes", as they have no known descendants at read-time. In formal terms, these are "causal sources". These nodes are merely subjectively the most recent fact: it is possible that some other facts were written elsewhere, but simply have not arrived at the reader yet. Since more facts MAY be appended to the history at any time.

There MUST be at least one head node in an inhabited graph (even if it is a single-node graph). There MAY be an unbounded number of concurrent head nodes.

#### 4.2.2.3 Concurrent Nodes

When comparing any two nodes, if one does not appear in the causal history of the other, then they are said to be "concurrent" or "sibling" nodes. As any two causally disjoint graphs have (by definition) no dependencies on each other, most facts are concurrent.

#### 4.2.2.4 Parent Nodes

Nodes listed in the [`Cause`] field of a fact are said to be that fact's "parent nodes". A fact MAY have zero or more parents.

``` mermaid
flowchart BT
    parentA
    parentB
    
    fact --> parentA
    fact --> parentB
```

#### 4.2.2.5 Child Nodes

Nodes that list a fact is their [`Cause`] field of a fact are said to be that fact's "child nodes". A fact MAY have zero or more children.

``` mermaid
flowchart BT
    childA --> fact
    childB --> fact
```

#### 4.2.2.6 Ancestors & Provenance

A complete causal history is built up by recursively following parent edges, from the node being investigated back to its geneses. As long as there is an unbroken path from one node to another, it is said to be the "descendant" of its "ancestor". For example, in the above graph, `bafy...avocado` is an ancestor of `bafy...baklava` along the blue path.

Only direct parents SHOULD be listed in a [`Cause`] field, as the complete history is intact [transatively][transative]. For example, in the graph above, `bafy...ambrosia` has no direct link to `bafy...agave` and `bafy...bun` has no direct link to `bafy...bean` because indirect, transative histories exists (shown in pink and orange respectively). The fact that this path crosses writers or stores is immaterial.

One exception to avoiding writing redundant links in a causal history when some of those ancestors are expected to have different visibility to peers, such as when some facts are encrypted. 

# 5 Prior Art

This is a large amount of prior art in this area. Below are a few resources that were either direct influences, or frequently brought up as comparisons to PomoDB. They are presented here alphabetically:

## 5.1 [Automerge]

Automerge is a nested CRDT that is capable of representing JSON. The Automerge team have developed many optimizations and demo applications for CRDTs in [Local-First] applications.

## 5.2 [Berkley Orders of Magnitude][BOOM]

The BOOM project has explored Datalogs for various distributed systems use cases for well over a decade. "Disorderly programming", the [CALM][Keeping CALM] theorem, and [Dedalus] have been major inspirations for PomoDB.

## 5.3 [Datasette]

Datasette is a tool for exploration, publishing, and managing relational data. A common use case for it is combining data from multiple sources and exploring it in rich interfaces.

## 5.4 [Datomic] & [Datahike]

Datomic was the first datalog experience for a lot of people. Datahike replicates a subset of the Datomic APIs, but is also able to run in the browser.

## 5.5 [Mentat]

Mentat is a client-side Datalog that strongly embraces schemaless design (though domain modeling is still encouraged).

> One might say that Mentat’s question is: “What if an SQLite database could store arbitrary relations, for arbitrary consumers, without them having to coordinate an up-front storage-level schema?”

## 5.6 [Project Cambria]

Cambria is a research project from [Ink & Switch] that attempts to solve JSON schema drift between different applications or versions. While Cambria uses lenses directly on JSON, PomoDB takes a lot of inspiration and expresses similar functionality as datalog rules.

## 5.7 [RDF]

The Resource Description Framework (or "RDF") is a mature data modeling and data store specification from the [W3C]. RDF has a very rich ecosystem of tools, and PomoDB draws from a lot of research based on RDF. It has been said that "PomoDB is RDF without formal ontologies and more CRDTs".

## 5.8 [Soufflé]

At time of writing, Soufflé is one of — if not "the" — premier extended Datalogs. It is very featureful, with numerous additions to overcome limitations in "classical" Datalog that are otherwise limitations for 

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
[Dedalus]: https://dsf.berkeley.edu/papers/datalog2011-dedalus.pdf
[Deleuze]: https://en.wikipedia.org/wiki/Gilles_Deleuze
[Double-precision]: https://en.wikipedia.org/wiki/Double-precision_floating-point_format#IEEE_754_double-precision_binary_floating-point_format:_binary64
[Eternalism]: https://en.wikipedia.org/wiki/Eternalism_(philosophy_of_time)
[Fallacies of distributed computing]: https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing
[Fission Codes]: https://fission.codes
[Guattari]: https://en.wikipedia.org/wiki/F%C3%A9lix_Guattari
[Hexastore]: https://people.csail.mit.edu/tdanford/6830papers/weiss-hexastore.pdf
[IPFS]: https://ipfs.io
[IPLD]: https://ipld.io/specs/
[Keep Calm and CRDT On]: https://www.vldb.org/pvldb/vol16/p856-power.pdf
[Keeping CALM]: https://arxiv.org/pdf/1901.01930.pdf
[Laurent Binet]: https://en.wikipedia.org/wiki/Laurent_Binet
[Local-First]: https://www.inkandswitch.com/local-first/
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
[W3C]: https://www.w3.org/
[Wasm Numbers]: https://webassembly.github.io/spec/core/syntax/types.html#syntax-numtype
[Wasm Opaque Reference Types]: https://webassembly.github.io/spec/core/syntax/types.html#reference-types
[Wasm Primitive Types]: https://webassembly.github.io/spec/core/appendix/index-types.html
[WebAssembly]: https://webassembly.org/
[WebNative File System]: https://github.com/wnfs-wg/spec
[`Attribute`]: #413-attribute
[`Cause`]: #416-cause
[`Entity ID`]: #412-entity-id
[`Value`]: #414-value
[content addressing]: #24-content-addressing
[ink & Switch]: https://www.inkandswitch.com/ 
[relation]: #22-relation
[serialization]: ./pomo_db/serialization.md
[sinks]: #28-sinks
[sources]: #27-sources
[time]: #22-time
[transative]: https://en.wikipedia.org/wiki/Transitive_relation
