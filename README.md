# Web-Native Inter-Agent Negotiation: An Illustrative Example

This document provides a concrete illustration of how the architectural primitives of the **Federated Personal Agents** ecosystem—agent identity, decentralized discovery, delegated authorization, and semantic messaging—compose to support inter-agent negotiation without exposing private user data.

The example focuses on a simple scheduling interaction between two users, **Alice** and **Bob**, mediated by their respective personal agents.

![Figure 1: High-Level Architecture of the Sovereign Federated Personal Agent System.](diagrams/solidfl-architecture.png)

---

## 1. Agent Discovery via Web-Native Identity

Agent discovery builds directly on existing Solid and Web identity mechanisms. In Alice’s primary Solid profile, she advertises the existence of her **Scheduling Agent** by linking to its WebID. This indirection enables other agents to discover Alice’s agent without accessing any of her private data.

### Listing 1: Alice's Profile (Turtle)

```turtle
@prefix foaf: <http://xmlns.com/foaf/0.1/> .
@prefix solid: <http://www.w3.org/ns/solid/terms#> .
@prefix myagents: <https://alice.solidweb.org/ontology/agents#> .

<#me> a foaf:Person ;
    foaf:name "Alice" ;
    # Discovery link to her specialized agent
    myagents:hasSchedulingAgent <https://alice.solidweb.org/agents/scheduling#id> .

```

Resolving the agent’s identifier yields a machine-readable profile document describing the agent’s role, identity provider, and communication endpoints. The profile below illustrates how the agent exposes its **Inbox** using standard linked data conventions, enabling asynchronous message delivery without requiring a centralized broker.

### Listing 2: Agent Profile Document

```turtle
<#id> a myagents:SchedulingAgent ;
    foaf:name "Alice's Chief of Staff" ;
    solid:oidcIssuer <https://alice.solidweb.org/> ; # Identity provider
    ldp:inbox <https://alice.solidweb.org/inbox/scheduling/> . # Message inbox

```

---

## 2. Delegated Authorization as a Capability Pattern

Inter-agent communication is governed by explicit, narrowly scoped authorization policies. Rather than granting Bob (or Bob’s agent) access to Alice’s calendar data, Alice delegates permission for Bob’s Scheduling Agent to **submit proposals** to her agent’s inbox.

This delegation is expressed using **Web Access Control (WAC)**:

### Listing 3: Access Control List (ACL) Entry

```turtle
@prefix acl: <http://www.w3.org/ns/auth/acl#> .

<#BobAgentAccess>
    a acl:Authorization ;
    acl:accessTo <https://alice.solidweb.org/inbox/scheduling/> ;
    acl:mode acl:Append ;
    acl:agent <https://bob.solidweb.org/agents/scheduling#id> .

```

We refer to this authorization pattern as the **Inbox-Append Delegation Pattern**. It represents a reusable Web-native capability for inter-agent interaction with the following properties:

* **Minimal disclosure:** The delegatee gains no access to underlying personal data or previously submitted messages.
* **Append-only messaging:** Interaction is constrained to message submission, preventing retrospective inspection or data leakage.
* **Revocability at the pod:** Authorization can be withdrawn unilaterally by the user without coordination with external agents.
* **Auditable trace:** Messages and authorization policies are stored as RDF resources, producing a durable, inspectable interaction record.

This pattern enforces a clear separation between *interaction* and *data access*, enabling negotiation and coordination without disclosure of private context.

---

## 3. Semantic Proposal Exchange

Once authorized, Bob’s Scheduling Agent initiates a negotiation by sending a semantic proposal to Alice’s agent inbox.

The payload below is encoded using **ActivityStreams** and **iCalendar** vocabularies. Rather than issuing an imperative command, the message represents an *intent*, a meeting invitation with associated temporal metadata.

### Listing 4: Incoming Proposal Payload

```http
POST /inbox/scheduling/ HTTP/1.1
Content-Type: text/turtle
From: <https://bob.solidweb.org/agents/scheduling#id>

@prefix as: <https://www.w3.org/ns/activitystreams#> .
@prefix ical: <http://www.w3.org/2002/12/cal/ical#> .

<urn:uuid:1234> a as:Invite ;
    as:actor <https://bob.solidweb.org/agents/scheduling#id> ;
    as:object [
        a ical:Vevent ;
        ical:summary "Project Sync" ;
        ical:dtstart "2026-01-04T14:00:00Z" ;
        ical:duration "PT1H"
    ] .

```

This semantic representation allows Alice’s agent to interpret the proposal independently of Bob’s internal logic and to reason over it using local context, policies, and preferences.

---

## 4. Local Reasoning & Outcome Disclosure

Upon receiving the proposal, Alice’s agent executes the following workflow:

1. **Verifies** the sender’s identity (WebID).
2. **Checks** authorization policies.
3. **Retrieves** context (e.g., calendar availability, wellness constraints) via the Model Context Protocol (MCP).

**Decision-making remains entirely local.** Bob’s agent observes only the negotiated outcome, not the private data or intermediate inferences.

Alice’s agent responds by posting a semantically linked reply to Bob’s agent inbox.

### Listing 5: Response Payload

```http
POST /inbox/scheduling/ HTTP/1.1

@prefix as: <https://www.w3.org/ns/activitystreams#> .

<urn:uuid:5678> a as:Reject ;
    as:inReplyTo <urn:uuid:1234> ;
    as:actor <https://alice.solidweb.org/agents/scheduling#id> ;
    as:summary "Alice is unavailable due to wellness blocks." ;
    as:object <urn:uuid:1234> .

```

Because both proposals and responses are represented as signed RDF resources and persisted within the respective users’ pods, the interaction produces a **durable, auditable trail**. This supports post hoc inspection and compliance verification without reliance on centralized platform logs.

---
