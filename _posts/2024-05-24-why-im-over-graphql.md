---
layout: post
title: "Why, after 6 years, I’m over GraphQL"
date: 2024-05-24 13:01
comments: true
categories: ruby rails graphql rest openapi
---

GraphQL is an incredible piece of technology that has captured a lot of mindshare since I first started slinging it in production in 2018. You won’t have to look far back on this (rather inactive) blog to see I have previously championed this technology. Coming from a rough and ready untyped hodge podge of REST APIs, I found GraphQL a breath of fresh air. I was truly a GraphQL hype train member.

However, as the years have gone on and I have had the opportunity to deploy to environments where non functional requirements like security, performance, and maintainability were more of a concern, my perspective has changed. In this article I would like to take you through why today, **I would not recommend GraphQL to most people**.

Throughout I will use Ruby code with the excellent [graphql-ruby](https://graphql-ruby.org/) library for examples, but I believe many of these problems are ubiquitous across choice of language / GraphQL library.

If you know of better solutions and mitigations, please do leave a comment. Now, lets begin…

# Attack surface

It was obvious from the beginning that exposing a **query language** to untrusted clients increases the attack surface of the application. Mitigating the wide variety of attacks effectively is significantly harder than I appreciated in the beginning. Lets consider a few…

## Authorisation

I think this is the most widely understood risk of GraphQL, so I won’t go into too much depth here. TLDR: if you expose a fully self documenting query API to all clients, you better be damn sure that every **field** on every type is authorised against the current user appropriately to the context in which that field is being fetched. Initially authorising **objects** seems like enough, but this quickly becomes insufficient. For example, say we are the ~~Twitter~~ X 🙄 API:

```graphql
query {
  user(id: 321) {
    handle # ✅ I am allowed to view Users public info
    email # 🛑 I shouldn't be able to see their PII just because I can view the User
  }
  user(id: 123) {
    blockedUsers {
      # 🛑 And sometimes I shouldn't even be able to see their public info,
      # because context matters!
      handle
    }
  }
}
```

One wonders how much GraphQL holds responsibility for Broken Access Control climbing to the [OWASP Top 10’s #1 spot](https://owasp.org/Top10/)… Compare this to the REST world where you would authorise **once** in the `/users/:id/blocked-users` endpoint with full knowledge of what checks need to be made for this endpoint’s response**.**

## Rate limiting

Rate limiting a REST endpoint is simple. You probably aren’t building endpoints that are orders of magnitude slower than your average response time, so a blanket bucketed rate limiter that prevents a user exceed 200 requests per minute across all endpoints is all you need. If you **do** have a few super slow endpoints (say, a CSV report or PDF generator) you can define additional more aggressive rate limits for these. With some HTTP middleware this is pretty trivial:

```ruby
Rack::Attack.throttle('API v1', limit: 200, period: 60) do |req|
  if req.path =~ '/api/v1/'
    req.env['rack.session']['session_id']
  end
end
```

With GraphQL however, we cannot assume that all requests are equally hard on the server. There is no limit to how big a query can be. Even in a completely empty schema, the types exposed for [introspection](https://graphql.org/learn/introspection/) are cyclical, so its possible to craft a valid query that returns MBs of JSON:

```graphql
query {
  __schema{
    types{
      __typename
      interfaces {
        possibleTypes {
          interfaces {
            possibleTypes {
              name
            }
          }
        }
      }
    }
  }
}
```

I just tested this attack against a **very** popular website’s GraphQL API explorer and got a 500 response back after 10 seconds. I just ate 10 seconds of someone’s CPU time running this (whitespace removed) **128 byte** query.

A common mitigation[1] for this attack is to

1. Estimate the complexity of resolving **every single field in the schema**, and abandon queries that exceed some maximum complexity value
2. Capture the actual complexity of the run query and take it out of bucket of credits that resets at some interval

This calculation is a [delicate affair to get right](https://graphql-ruby.org/queries/complexity_and_depth#prevent-complex-queries). It gets particularly tricky when you are returning list fields whose length is not known prior to execution. In these cases you can make an assumption about the length of these, but you better hope an attacker cannot find a case where your estimate is low, because…

## Query recursion

Its common for the graph that makes up the schema to contain cycles. Lets say you run a blog with Articles which each have multiple Tags, from which you can see associated Articles.

```graphql
type Article {
  title: String
  tags: [Tag]
}
type Tag {
  name: String
  relatedTags: [Tag]
}

query {
  tag(name: "security") {
    relatedTags {
      relatedTags {
        relatedTags {
          relatedTags {
            relatedTags { name }
          }
        }
      }
    }
  }
}
```

When you set up the rate limiting system, you assume that an article will never have more than 5 tags, so you set this fields complexity to **5** (or 5 * [its children’s complexity](https://graphql-ruby.org/queries/complexity_and_depth#how-complexity-scoring-works)). Do you see the problem? `Article.tags` is its own child, so your estimate’s inaccuracy compounds exponentially. The formula is `N^5 * 1` , so you expect this query to be of complexity `5^5 = 3,125` so you set your max complexity to 5,000. If an attacker is able to find an Article with 10 tags, they can trigger a query that’s complexity is 10^5 is **100,000,** 20x greater than you intended.

A partial mitigation here is to [prevent deeply nested queries](https://graphql-ruby.org/queries/complexity_and_depth#prevent-deeply-nested-queries). However, the example above demonstrates that this is not really a defense. Its’ pretty common to see 7 level deep queries in my experience. GraphQL Ruby’s default is **13.**

## Query parsing

Before a query is executed, it is first parsed. We once received a pen-test report evidencing that its possible to craft an invalid query string that OOM’d the server. For example:

```graphql
query {
  __typename @a @b @c @d @e ... # imagine 1k+ more of these
}
```

This is a **syntactically** valid query, but invalid for our schema. A spec compliant server will parse this and start building an errors response containing thousands of errors which consume far more memory than the query string itself.

If your server exposes a concept of max errors before abandoning parsing, [this can be mitigated](https://github.com/rmosolgo/graphql-ruby/issues/4154), but its yet another GraphQL specific attack to worry about.

# Performance

When it comes to performance in GraphQL people often talk about it’s incompatibility with HTTP caching. For me personally, this has not been an issue. For SaaS applications, data is usually highly user specific and serving stale data is unacceptable, so I have not found myself missing response caches (or the cache invalidation bugs they cause…).

The performance problems I **did** find myself bashing my head against were…

## Data fetching and the N+1 problem

I think this issue is pretty widely understood nowadays. TLDR: if a field resolver hits an external data source such as a DB or HTTP API, and it is nested in a list containing N items, it will do those calls N times.

This is not a unique problem to GraphQL, and actually the strict GraphQL resolution algorithm has allowed most libraries to share a common solution: [the Dataloader pattern](https://graphql-ruby.org/dataloader/overview.html). Also unique to GraphQL is the fact that, again, since we’re a query language, this can **become** a problem with no backend changes when a client modifies a query. As a result, you end up having to introduce the Dataloader abstraction everywhere *just in case* a client ends up fetching a field in a list context in the future.

Meanwhile, in REST, we can generally hoist nested N+1 queries up to the controller, which I think is a pattern much easier to wrap your head around:

```ruby
class BlogsController < ApplicationController
  def index
    @latest_blogs = Blog.limit(25).includes(:author, :tags)
    render json: BlogSerializer.render(@latest_blogs)
  end

  def show
    # No prefetching necessary here since N=1
    @blog = Blog.find(params[:id])
    render json: BlogSerializer.render(@blog)
  end
end
```

## Authorisation and the N+1 problem

But wait, there’s more N+1s! Lets say you’ve been a careful dev and ensured your API is secure by default by integrating with [your library’s authorisation framework](https://graphql-ruby.org/authorization/authorization). Every object returned and/or field resolved, your authorisation system is called to confirm that the current user has access. Well, you’ve now got a whole new category of N+1 problems to deal with. Lets continue with our X API example from earlier:

```ruby
class UserType < GraphQL::BaseObject
  field :handle, String
  field :birthday, authorize_with: :view_pii
end

class UserPolicy < ApplicationPolicy
  def view_pii?
    # Oh no, I hit the DB to fetch the user's friends
    user.friends_with?(record)
  end
end
```

```graphql
query {
  me {
    friends { # returns N Users
      handle
      birthday # runs UserPolicy#view_pii? N times
    }
  }
}
```

This is actually trickier to deal with than our previous example, because authorisation code is not always run in a GraphQL context, it may for example be run in a background job or an HTML endpoint. That means we can’t just reach for a Dataloader naively, because Dataloaders expect to be run from within GraphQL (in the Ruby implementation anyway).

In my experience, this is actually **the biggest source of performance issues,** and again, one that simply does not exist in the REST world. I have mitigated this using nasty things like [request level globals](https://api.rubyonrails.org/classes/ActiveSupport/CurrentAttributes.html) to memoise data across policy calls, but its never felt great.

# Coupling

In my experience, in a mature GraphQL codebase, your business logic is forced into the transport layer. This happens through a number of mechanisms, some of which we’ve already talked about:

- Solving data authorisation leads to peppering authorisation rules throughout your GraphQL types
- Solving N+1 resolver data fetching leads to moving this logic into GraphQL specific dataloaders
- Leveraging the (lovely) [Relay Connection](https://graphql-ruby.org/pagination/using_connections) pattern leads to moving data fetching logic into GraphQL specific [custom connection objects](https://graphql-ruby.org/pagination/custom_connections)

The net effect of all of this is to meaningfully test your application you **must** extensively test at the integration layer, i.e. by running GraphQL queries. I have found this makes for a much worse experience. Errors in field resolution are captured by the framework, leading to the fun task of reading stack traces in JSON responses. Since so much around authorisation and Dataloaders happens inside the framework, debugging is often much harder as the breakpoint you want is not in application code.

And of course, again, since its a query language you’re going to be writing a lot more tests to confirm that all those field level behaviours we mentioned are working correctly.

# Complexity

Taken in aggregate, the various mitigations to security and performance issues we’ve gone through add **significant** complexity to a codebase. Over the years I have become far more knowledge on the GraphQL Ruby library than I would care to be, simply because I have had to debug so many stack traces that have woven in and out of GraphQL framework code and application code, in a way that I did not see in the REST world.

# And more…

That concludes what for me are the biggest things I wish I’d been warned about as GraphQL. As a quick fire round of other points…

- GraphQL discourages breaking changes and provides no tools to deal with them. This adds needless complexity for those who control all their clients, who will have to [find workarounds](https://graphql-ruby.org/changesets/overview.html).
- The semantic meaning of HTTP response codes are everywhere, and dealing with the fact that 200 can mean everything from everything is Ok through to everything is down can be quite annoying.
- Fetching all your data in one query in the HTTP2+ age is often not beneficial, in fact it will worsen response time if your server is not parallelised, vs sending separate requests to separate servers .

# So now what?

To be up front, I am definitely early in the [hype cycle](https://en.wikipedia.org/wiki/Gartner_hype_cycle) here, but right now my view is that if you:

1. Control all your clients
2. Have ≤3 clients
3. Have a client written in a statically typed language
4. Have a mixture of languages across server and clients[2]

You are probably better off reaching for an **OpenAPI 3.1** compliant JSON REST API.

Tooling in this area has improved a lot since GraphQL came on the scene. There are many options for generating typed client code even down to [framework specific data fetching libraries](https://orval.dev/overview).

For Python backends, [FastAPI](https://fastapi.tiangolo.com/) has pioneered generating a typed schema from Python type hints alone. That is what I have been loving lately. My experience so far is pretty close to “the best parts of what **I** used GraphQL for, without the complexity Facebook needed”.

For those working in statically typed backend languages, the recent introduction of [TypeSpec](https://typespec.io/) looks to be particularly interesting. With it, perhaps in the future your workflow could be:

1. Write a succinct human readable TypeSpec schema
2. Generate an OpenAPI YAML spec from it
3. Generate statically typed client handlers in your frontend language of choice (e.g. [JS / TS](https://orval.dev/overview))
4. Generate statically typed server handlers in your backend language of choice (e.g. [Go](https://github.com/deepmap/oapi-codegen))
5. Write an implementation [for that handler](https://github.com/deepmap/oapi-codegen#echo-server) that compiles, using whatever patterns internally you prefer

To me, it seems like powerful **and** simpler options are here, and I’m excited to learn their drawbacks 😄

[1] Persisted queries are also a mitigation for this and many attacks, but if you actually want to expose a customer facing GraphQL API, persisted queries are not an option.

[2] Otherwise a language specific solution like [tRPC](https://trpc.io/) might be a better fit.
