---
layout: post
title:  "GraphQL Is Great (Until It Isn't)"
date:   2026-02-18 10:00:00
---

Everyone talks about how GraphQL solves over-fetching, enables flexible clients, and gives you a beautiful self-documenting schema. And yes, it does all of that. But after spending enough time building real systems with it, you start accumulating a collection of quiet frustrations. The kind that don't make it into the tutorials.

Let's talk about them honestly.

---

## The N+1 Problem (The One Everyone Knows)

This is GraphQL's most well-documented pitfall, so let's get it out of the way.

Imagine a `users` query that returns a list of users, each with a `posts` field. A naive resolver looks like this:

```graphql
query {
  users {
    name
    posts {
      title
    }
  }
}
```

Under the hood, you fetch `N` users, then fire `N` separate database queries to get posts for each one. So if you have 100 users, you just made 101 queries. Congratulations.

The canonical fix is **DataLoader**, a batching and caching utility that collects all the IDs within a single tick of the event loop and resolves them in one shot. It works, but it means every resolver that touches a relationship now needs a DataLoader instance. That's a lot of boilerplate, and it needs to be scoped per-request to avoid cross-user data leaks (yes, that's a real footgun people hit in production).

The deeper issue: **GraphQL doesn't make N+1 a compile-time concern.** There's no schema-level annotation that says "this field is expensive." Junior developers write resolvers that look correct and work fine in development, then destroy the database in production when real data volumes arrive.

---

## Schema Design Is Deceptively Hard

REST APIs are coarse-grained by nature. Endpoints are nouns with verbs. GraphQL invites you to expose *everything*, which means you need to make real decisions about your domain model.

Do you put `viewer` as a root query field or have every query accept a `userId`? Do mutations return the affected object or a payload wrapper? Do you use connections (edges/nodes) for all lists, or only paginated ones? Do you put business logic in resolvers or keep them thin?

These aren't GraphQL problems per se, but GraphQL *amplifies* the cost of getting them wrong. REST endpoints are cheap to refactor because they're isolated. GraphQL schema types are interconnected. A bad naming decision for `User` bleeds into every type that references it.

The community has conventions (Relay spec, etc.), but they're not enforced. Teams standardize inconsistently, and suddenly your schema has `User`, `Author`, `Member`, and `AccountHolder` that are all sort of the same thing.

---

## Error Handling Is a Mess

REST has HTTP status codes. `400` means bad input. `404` means not found. `500` means something exploded. Every client, proxy, monitoring tool, and cache understands these.

GraphQL always returns `200 OK`. Even when things go wrong.

The error model looks like this:

```json
{
  "data": { "user": null },
  "errors": [{ "message": "User not found", "path": ["user"] }]
}
```

Now ask yourself: how does your CDN know not to cache this? How does your APM tool alert on elevated error rates by default? How does your mobile client know to show a "not found" screen vs. a "server error" screen?

You have to build all of this yourself. Error codes in extensions, custom error types, union result types (`type UserResult = User | NotFoundError | AuthError`). All workarounds for a missing primitive.

Union result types are actually the most ergonomic solution, but they require buy-in from your entire team, and they make every mutation feel verbose:

```graphql
mutation CreatePost($input: CreatePostInput!) {
  createPost(input: $input) {
    ... on Post { id title }
    ... on ValidationError { field message }
    ... on AuthError { message }
  }
}
```

That's a lot of ceremony for "did this work?"

---

## Authorization Logic Has No Good Home

With REST, a middleware chain is natural. Auth checks happen before the request touches business logic. With GraphQL, you have one endpoint and resolvers for everything. Where does authorization live?

Options:

**In resolvers** work, but you end up copy-pasting auth checks everywhere and praying someone doesn't forget one.

**Schema directives** like `@auth(role: ADMIN)` sound elegant but require a non-trivial amount of infrastructure and don't compose well with complex permission logic.

**A dedicated authorization layer** via libraries like `graphql-shield` lets you write permission rules separately, but they add complexity and debugging when permissions misfire can be opaque.

The real problem is that GraphQL queries can be arbitrarily nested and composed. A user might be allowed to read `Order`, but not allowed to read `Order.customer.email`. Field-level permissions in a deeply nested graph are genuinely hard, and there's no blessed solution.

---

## Introspection Is a Security Concern Nobody Talks About

GraphQL's killer feature, the self-documenting schema, is also a blueprint for attackers. A single introspection query reveals every type, every field, every argument, every relationship in your API.

Most GraphQL APIs ship to production with introspection enabled because it's on by default and people forget to disable it. If you're exposing a public API, you're handing a complete map of your data model to anyone curious enough to ask.

The fix is simple (disable introspection in production, or gate it behind auth), but the fact that it's a footgun people routinely fall into is worth naming.

---

## Query Complexity and Depth Are Your Problem

GraphQL allows clients to request *anything they want*, which is kind of the point. It also means a malicious or just-careless client can send a query like:

```graphql
{
  users {
    friends {
      friends {
        friends {
          friends { name }
        }
      }
    }
  }
}
```

This is an exponentially expensive query that REST would never have let through because the endpoint just doesn't exist. With GraphQL, it does, and it'll happily try to execute it.

You need to explicitly implement query depth limits and complexity scoring. Neither is on by default. Both require you to assign complexity weights to fields, which is manual, error-prone work that nobody updates when the schema evolves.

---

## File Uploads Are an Afterthought

The GraphQL spec says nothing about file uploads. If you need to upload an image alongside some metadata, you're choosing between:

- A completely separate REST endpoint for the upload (practical but feels wrong)
- The `multipart/form-data` spec extension, which requires middleware support that isn't universal and causes CORS headaches
- Base64 encoding the file in the mutation (fine for small files, a disaster for anything real)

It's a genuine gap that the ecosystem papers over imperfectly.

---

## Caching Is Fundamentally Harder

HTTP caching is essentially free with REST. GET requests are cacheable by default, CDNs understand them natively, and ETag/Last-Modified headers just work. GraphQL ships almost entirely over POST, which is not cached by anything by default.

You can use persisted queries to convert queries into GET-able hashes, but that requires tooling on both client and server, buy-in from every client team, and a registry to keep in sync.

Client-side normalized caches (Apollo Client, urql) are impressive engineering, but they're complex, they have subtle bugs around cache invalidation, and they're essentially reimplementing what HTTP already gives you for free on REST.

---

## The Tooling Tax

GraphQL's developer experience (GraphiQL, schema introspection, typed client generation) is genuinely excellent. But it requires infrastructure that REST doesn't.

You need a schema registry. You need to run codegen after every schema change. You need to lint your schema. You need to configure your IDE plugin. You need to think about schema stitching or federation if you have multiple services. You need Apollo Studio or a comparable service if you want operation metrics.

None of this is hard exactly, but it's weight. A team that's already stretched building product features will feel it.

---

## When GraphQL Is Worth It

None of this is an argument against GraphQL. For client-heavy applications like mobile apps and SPAs where different views need wildly different data shapes, GraphQL's flexibility genuinely earns its keep. For public APIs where you want to give third-party developers expressive querying power, it's excellent.

But it's not a free lunch, and the ecosystem has a tendency to undersell the operational cost. The N+1 problem is just the most visible tip of a larger iceberg. A query language powerful enough to let clients ask for anything puts the burden of *everything they shouldn't ask for* firmly on the server team.

Know what you're getting into. DataLoader your resolvers. Disable introspection in production. Limit query depth. Design your schema like it's forever, because with GraphQL, it basically is.