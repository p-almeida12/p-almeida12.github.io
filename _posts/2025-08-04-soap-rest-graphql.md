---
layout: post
title: SOAP, REST, and GraphQL - My experience
date: 2025-08-04 10:59:00-0400
description: SOAP, REST, and GraphQL - trying to compare the uncomparable.
tags:
categories:
giscus_comments: false
related_posts: false
toc:
  beginning: true
---
<span style="margin-left: 10px;"></span>
APIs define how services talk, how frontends fetch data, and ultimately how systems scale.
Throughout my career, I’ve spent most of my time working with REST APIs in enterprise contexts.

<span style="margin-left: 10px;"></span>
In a recent personal project with a friend, I migrated from REST to GraphQL (using Netflix DGS) to overcome 
data-fetching inefficiencies. I’ve also tinkered with SOAP, though never in production — enough to 
appreciate its design philosophy and understand why it still powers parts of the enterprise world.

<span style="margin-left: 10px;"></span>
This post is my deep dive into SOAP, REST, and GraphQL — how they differ, how they compare (or don’t), and 
what lessons I’ve learned from working with them.

# Are SOAP, REST, and GraphQL Comparable?
<span style="margin-left: 10px;"></span>
At first glance, these three might seem like direct competitors — but they’re not perfectly comparable.

| **Comparison Aspect**      | **SOAP**                      | **REST**     | **GraphQL**                             |
|----------------------------|-------------------------------|-------------|-----------------------------------------|
| **Category**    | Communication protocol.       | Architectural style       | Query language.                         |
| **Level of abstraction**  | Low-level, rigid.             | Conceptual and flexible    | Declarative and client-driven.          |
| **Intent**         | Standardized data exchange.   | Resource-oriented access       | Flexible, client-specific data queries. |

<span style="margin-left: 10px;"></span>
In short:
- SOAP is a protocol — like a contractually defined tunnel between systems.
- REST is a style — a set of guiding principles for web-based resource communication.
- GraphQL is a language/runtime layered on top of HTTP — a tool to query data in a declarative, efficient way.

<span style="margin-left: 10px;"></span>
So while they all enable API communication, they do it at different levels of abstraction and serve 
different design goals. Comparing them might seem wrong, but its possible, we just need to keep in mind that 
they are different and optimized for different situations.

## SOAP

<span style="margin-left: 10px;"></span>
SOAP (Simple Object Access Protocol) was designed for interoperability in large, distributed systems. 
It enforces a strict structure using XML envelopes, WSDL contracts, and WS-extensions for things like 
security, transactions, and reliable messaging.

<span style="margin-left: 10px;"></span>
SOAP message example:

```
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Header>
    <auth:Token>VerYsECUreTOkeN124</auth:Token>
  </soap:Header>
  <soap:Body>
    <m:GetBeer xmlns:m="http://goodbeer.com/beers">
      <m:BeerId>42</m:BeerId>
    </m:GetBeer>
  </soap:Body>
</soap:Envelope>
```

### Architecture

<span style="margin-left: 10px;"></span>
SOAP follows a strictly layered and contract-driven architecture. It was designed to 
ensure interoperability between systems written in different languages and running 
on different platforms. The architecture is protocol-centric, with a focus on message 
structure, transport abstraction, and formal service contracts.

<span style="margin-left: 10px;"></span>
At its core, a SOAP system consists of:
- SOAP Envelope — the root XML element that defines the start and end of the message.
- Header — optional section that carries metadata such as authentication tokens, 
transaction IDs, or routing information.
- Body — contains the actual request or response payload.
- Fault Element — a special XML element used for standardized error reporting.

<span style="margin-left: 10px;"></span>
SOAP services are usually defined by a WSDL (Web Services Description Language) file. 
The WSDL acts as a contract that describes:
- The available operations (methods) and their parameters.
- The XML schema (XSD) defining the structure of input/output data.
- The binding details (which protocol and encoding are used).
- The service endpoint (where the service lives).

<span style="margin-left: 10px;"></span>
When a client consumes a SOAP service, it typically:
1. Parses the WSDL to generate client-side proxy classes.
2. Constructs a XML request message conforming to the schema.
3. Sends the message over a chosen transport (HTTP, SMTP, or others).
4. Receives an XML response and deserializes it into native objects.

<span style="margin-left: 10px;"></span>
Because SOAP is transport-independent, it can use multiple communication protocols. 
HTTP is the most common, but SOAP can also run over JMS (Java Message Service) for 
asynchronous enterprise communication.

<span style="margin-left: 10px;"></span>
This rigidity brings predictability and strong validation, but also makes SOAP 
services heavier and harder to evolve. Every change in schema or contract typically 
requires regenerating stubs and revalidating clients.

### Strengths and Weaknesses

| **Pros**                                                                            | **Cons**                    |
|-------------------------------------------------------------------------------------|-----------------------------|
| Strong typing and contract-first design with WSDL                                   | Verbose XML payloads increase complexity     |
| Built-in standards for security and reliability (WS-Security, WS-ReliableMessaging) | Harder to integrate with lightweight clients (web, mobile)           |
| Transport-agnostic (HTTP, SMTP, TCP, etc.)                                          | Tooling can feel heavy and outdated |
| Excellent for regulated industries (finance, telecom, insurance)                    | Difficult to evolve and version APIs flexibly |

<span style="margin-left: 10px;"></span>
When I experimented with SOAP, I found it, for lack of a better word, "elegant": you get 
guarantees, schemas, and rigid consistency. But in practice, the verbosity and 
dependency on specialized tooling (like JAX-WS or WSDL generators) made it feel 
awkward for modern distributed systems.

<span style="margin-left: 10px;"></span>
SOAP is a tank — extremely reliable but slow and rigid. For systems where every 
transaction must be traceable, it still earns its place. But for web APIs that 
need agility, it’s probably not my first choice.

## REST

<span style="margin-left: 10px;"></span>
REST (Representational State Transfer) redefined how we think about APIs. Instead of operations, REST focuses on 
resources — entities exposed through predictable URLs and standard HTTP verbs.

<span style="margin-left: 10px;"></span>
For example:

```
GET /beers/123
PUT /beers/123
DELETE /beers/123
```

<span style="margin-left: 10px;"></span>
REST emphasizes:
- Statelessness — each request contains everything needed.
- Uniform interface — clients and servers communicate via standard verbs.
- Cacheability — responses can be cached via HTTP.
- Layered system — intermediaries can optimize performance.

### Architecture

<span style="margin-left: 10px;"></span>
REST is an architectural style, not a protocol. Its architecture revolves around 
a set of constraints defined by Roy Fielding, which make web systems scalable, 
cacheable, and loosely coupled.

<span style="margin-left: 10px;"></span>
A RESTful system is organized around resources — conceptual entities exposed via URIs. 
Each resource can have multiple representations (usually JSON or XML) and is 
manipulated through standard HTTP verbs.

<span style="margin-left: 10px;"></span>
The key architectural constraints of REST are:
1. Client-Server Separation — clients handle UI and state, servers handle data and logic.
2. Statelessness — every request from the client must contain all information needed 
by the server. The server doesn’t store client session state.
3. Cacheability — responses must define themselves as cacheable or not, enabling 
intermediate caches (CDNs, proxies).
4. Uniform Interface — consistent behavior across all endpoints via HTTP verbs and 
standard response codes.
5. Layered System — clients don’t need to know whether they’re talking directly to 
the server or through intermediaries.
6. Optional Code on Demand — servers can return executable code (e.g., JavaScript) to 
clients.

<span style="margin-left: 10px;"></span>
A typical REST architecture looks like this:
- Client layer — browsers, mobile apps, or other services making HTTP calls.
- API Gateway / Load Balancer — handles routing, authentication, rate limiting.
- Application layer — REST controllers or handlers process requests, apply business logic, and access data sources.
- Persistence layer — databases or external systems providing the underlying data.

<span style="margin-left: 10px;"></span>
Data is exchanged through representations (usually JSON), with HATEOAS (Hypermedia as the Engine of Application 
State) being a powerful concept — embedding links in responses to guide clients through available 
actions.

<span style="margin-left: 10px;"></span>
In microservice environments, REST APIs are often interconnected through internal HTTP calls or API gateways like Kong, NGINX, or Spring Cloud Gateway.
Each microservice maintains its own resources, forming a loosely coupled ecosystem.

<span style="margin-left: 10px;"></span>
The architectural simplicity and alignment with web standards are what make REST so ubiquitous — 
but its statelessness and fixed endpoints can create inefficiencies when clients need complex, nested data.

<span style="margin-left: 10px;"></span>
I found this post by Fielding very clarifying and interesting: [https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven](https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven). 

### Strengths and Weaknesses

| **Pros**                                                                            | **Cons**                    |
|-------------------------------------------------------------------------------------|-----------------------------|
| Simple, human-readable, easy to test and debug                                  | Over-fetching: clients often get more data than needed     |
| Natively supported by web infrastructure | Under-fetching: multiple requests to build composite views           |
| Tooling and standards (OpenAPI, Swagger, Postman) are mature        | Versioning can be messy (/v1/, /v2/, etc.) |
| Easy integration with browsers, mobile apps                   | Limited flexibility for complex or nested data |
| Ideal for microservices and stateless systems                  | Can become chatty (many endpoints, round-trips) |

<span style="margin-left: 10px;"></span>
At work, REST is the backbone of our service-to-service communication. It’s pragmatic, reliable, and 
well-understood by everyone — from backend to frontend engineers.

<span style="margin-left: 10px;"></span>
However, over time I’ve fought the classic REST challenges, both at work and in my personal projects:
- Endpoints proliferating (/beer, /beer/details, /beer/details/extended).
- UIs needing multiple requests for a single view.
- The friction of maintaining backward compatibility when clients depend on specific payload shapes.

<span style="margin-left: 10px;"></span>
These trade-offs eventually led me to explore GraphQL for my personal projects.

## GraphQL

<span style="margin-left: 10px;"></span>
GraphQL represents a shift from resource-oriented design (REST) to data-oriented design.

<span style="margin-left: 10px;"></span>
Instead of defining how clients access resources, we define what data exists, and let clients query the 
exact shape they need.

<span style="margin-left: 10px;"></span>
Single endpoint: /graphql

<span style="margin-left: 10px;"></span>
Example query:

```
query {
  brewery(id: 1) {
    name
    beers {
      name
      reviews {
        text
      }
    }
  }
}
```
<span style="margin-left: 10px;"></span>
Server responds exactly with:

```
{
  "data": {
    "brewery": {
      "name": "Super Bock Brewery",
      "beers": [
        {
          "name": "Super Bock Original",
          "reviews": [
            { "text": "Smooth!" },
            { "text": "Perfect summer beer." }
          ]
        },
        {
          "name": "Stout",
          "reviews": [
            { "text": "Good." }
          ]
        }
      ]
    }
  }
}
```

### Architecture

<span style="margin-left: 10px;"></span>
GraphQL’s architecture flips the REST model by introducing a query engine between the client and the data sources. 
Instead of exposing multiple endpoints, GraphQL exposes a single endpoint that accepts declarative queries 
describing the exact shape of the data the client wants.

<span style="margin-left: 10px;"></span>
At a high level, a GraphQL system consists of:
- Schema Definition Layer — written in SDL (Schema Definition Language), describing types, queries, mutations, 
and relationships.
- Execution Layer (Resolvers / Data Fetchers) — functions that fetch data for specific fields or types.
- Query Engine — parses, validates, and executes incoming queries, orchestrating calls to resolvers.
- Data Sources Layer — the data providers

<span style="margin-left: 10px;"></span>
Here’s how a typical query flow works:
1. The client sends a JSON payload with a query or mutation to /graphql.
2. The GraphQL server parses the query and validates it against the schema.
3. The query engine executes each field’s resolver, often asynchronously.
4. Data from various sources (SQL, REST APIs, caches) is merged into the requested shape.
5. The server returns exactly the requested fields in JSON.

<span style="margin-left: 10px;"></span>
Resolvers are small, composable units that describe how to fetch a specific piece of data.
This makes GraphQL naturally extensible and federated — different services can own parts of the schema and 
resolve their respective data independently.

<span style="margin-left: 10px;"></span>
Because of this architecture, GraphQL excels at aggregating data from multiple backend sources into a single 
queryable interface.
It trades some server-side simplicity for client flexibility and performance.
While REST structures the world by endpoints, GraphQL structures it by types and relationships.

### Strengths and Weaknesses

| **Pros**                                             | **Cons**                    |
|------------------------------------------------------|-----------------------------|
| Fetch exactly what you need (no over/under-fetching) | Complexity in caching and monitoring     |
| Single endpoint simplifies API evolution             | Initial learning curve for schema & resolver patterns           |
| Self-documenting schema (introspection)              | Potential N+1 problem in naive resolvers |
| Excellent for frontend-driven development            | Overhead for simple CRUD use cases |
| Strong tooling ecosystem (DGS and many others)       | Requires runtime execution layer, not just HTTP routing |

### The N+1 Problem — and the DGS Solution

<span style="margin-left: 10px;"></span>
In my personal project, I initially ported a REST backend to GraphQL to reduce the 
multiple HTTP calls needed for the same thing in different views. The benefits were immediate — fewer 
endpoints, cleaner frontend queries, and faster perceived performance.

<span style="margin-left: 10px;"></span>
But then I hit the N+1 query problem — where nested resolvers cause multiple 
database hits per entity. For example, fetching a list of breweries with their beers 
triggered one DB query per brewery.

<span style="margin-left: 10px;"></span>
Thankfully, using Netflix DGS, I leveraged its DataLoader integration — a batching and 
caching mechanism that consolidates those queries into efficient bulk operations.
It was a game changer, turning what could have been a performance regression into a 
well-optimized data flow.

<span style="margin-left: 10px;"></span>
Netflix DGS (Domain Graph Service) is an open-source GraphQL framework built by 
Netflix and designed to simplify GraphQL server development in Java and Kotlin. 
It’s opinionated in a good way — integrating tightly with Spring Boot, enforcing 
schema-first development.

<span style="margin-left: 10px;"></span>
The Netflix DGS framework implements this architecture within the Spring ecosystem. It follows a schema-first
approach:
- You start by defining .graphqls schema files.
- Then implement @DgsQuery, @DgsMutation, and @DgsData components that act as resolvers.
- DGS automatically wires them into a GraphQL runtime, handling execution, validation, and DataLoader batching.

<span style="margin-left: 10px;"></span>
Internally, DGS uses graphql-java as the core engine and adds Netflix’s tooling for:
- Schema Federation (composing multiple domain services into one unified graph).
- Instrumentation and metrics collection.
- Schema registry integration and validation pipelines.
- DataLoader integration to batch and cache nested queries efficiently.

<span style="margin-left: 10px;"></span>
Unlike some other GraphQL implementations, DGS treats the GraphQL schema 
(.graphqls files) as the source of truth.
From that schema, you implement data fetchers — the functions that resolve the 
data for each type or field.

<span style="margin-left: 10px;"></span>
For example:

```
type Brewery {
  id: ID!
  name: String!
  beers: [Beer]
}

type Beer {
  id: ID!
  name: String!
  reviews: [Review]
}

type Query {
  brewery(id: ID!): Brewery
}
```

<span style="margin-left: 10px;"></span>
What I really appreciate about DGS is how it enforces structure without removing 
flexibility.
Resolvers are just annotated functions, so it fits neatly into Spring Boot’s 
dependency injection ecosystem.
You also get built-in integration with DataLoader, which helps solve the N+1 
query problem by batching and caching database calls.

<span style="margin-left: 10px;"></span>
Another big plus is how well DGS plays with GraphQL Federation.
In a larger setup, multiple teams can build their own GraphQL services 
(each describing a “domain”), and Netflix DGS provides tools to stitch these 
schemas together.

## Final Thoughts

<span style="margin-left: 10px;"></span>
SOAP, REST, and GraphQL are not direct competitors, but evolutionary stages in how we 
exchange data:
- SOAP gave us rigor — at the cost of agility.
- REST gave us simplicity — at the cost of precision.
- GraphQL gives us precision — at the cost of complexity.

<span style="margin-left: 10px;"></span>
In my professional work, REST has been the backbone of countless microservices. It’s consistent, predictable, and 
incredibly well-understood across teams. The tooling is mature, the caching model is natural, and HTTP’s semantics 
fit like a glove. But REST also taught me its limits — especially in frontend-heavy applications where data rarely 
fits neatly into single resources. I’ve seen endpoints multiply, clients chained across several requests, and 
payloads balloon to cover every edge case.

<span style="margin-left: 10px;"></span>
That pain is exactly what led me, in a personal project with a friend, to experiment with GraphQL. The immediate 
payoff was huge: fewer round trips, cleaner frontend code, and a noticeable reduction in friction between backend 
and UI. 

<span style="margin-left: 10px;"></span>
But the joy was brief. As our schema grew and queries became more nested, we ran headlong into the N+1 query 
problem — a classic GraphQL problem where resolvers trigger multiple database hits per record. Performance 
went down the drain. The problem wasn’t GraphQL itself; it was our implementation.

<span style="margin-left: 10px;"></span>
That’s when we discovered Netflix DGS.
DGS reframed GraphQL for us — not as an experimental tool, but as an enterprise-ready framework. Its schema-first 
approach gave structure where GraphQL alone offered freedom, and its DataLoader integration handled batching so 
elegantly that our N+1 nightmare disappeared almost overnight.
Suddenly, GraphQL development felt disciplined again — type-safe, introspectable, and scalable.

<span style="margin-left: 10px;"></span>
If SOAP is the tank, REST the reliable pickup truck, GraphQL is the modern EV — 
sleek and efficient, but requiring a bit more engineering under the hood to reach its potential.

<span style="margin-left: 10px;"></span>
The lesson?
Each has its place — the key is knowing when to drive which one.

