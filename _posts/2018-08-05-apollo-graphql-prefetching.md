---
layout: post
title: "Pre-fetching Data for Apollo GraphQL Client"
date: 2018-08-05 13:01
comments: true
categories:
  - ruby
  - rails
  - react
  - apollo client
  - graphql
---

**UPDATE: This post has been made redundant by my discovery of a simpler approach. Please see: [Server Side Rendering for React + Apollo GraphQL Client]({% post_url 2019-01-02-apollo-graphql-hypernova %}).**

In a previous post I talked about how I set up [server side rendering of React components in Rails with Hypernova]({{ site.baseurl }}{% post_url 2018-08-04-rails-webpacker-react-ssr %}). One detail I skimmed over in that post was _how_ I got the data dependencies of our React components in Ruby-land. We're a React + GraphQL shop (via [Apollo Client](https://github.com/apollographql/apollo-client)), and following GraphQL best practices, our data requirements are [colocated with the components that need the data](https://www.apollographql.com/docs/react/advanced/fragments.html#colocating-fragments), so this is a pretty tough problem. At first this seemed unsolvable, but thanks to some amazing tooling from the Apollo team, it can be done with surprising ease, and best of all no code duplication.

In this post I will walk you through how to extract your React components' GraphQL queries, fragments and all, into a format a Rails app can read, run, enabling full data hydration and server side rendering. While I am using Rails and React, **the principles of this tutorial can be applied to any server language with a GraphQL client, and to any JS framework using Apollo Client** or (I believe) any GraphQL JS client that supports denoting queries with the `gql` tag.

<!-- more -->


The fantastic tool we'll be using to accomplish this is [apollo-cli](https://github.com/apollographql/apollo-cli), a multipurpose tool for GraphQL code generation and schema analysis. This tool is nuts, allowing you to do all sorts of interesting stuff, but what we're interested in is its ability to extract queries from a inline `gql` tags in a React component and `.graphql` files, into a JSON file easily read by another service.

### Extracting Your Client Queries

To get started, lets install the CLI:

```bash
yarn add dev apollo-cli
```

Now we need to save the GraphQL schema that Apollo is querying, as the the CLI's code-gen functionality relies on it. This is achieved with GraphQL's [powerful introspection tooling](https://graphql.org/learn/introspection/). You may well already be doing this some other way, as it is a common requirement of lots of interesting GraphQL tools. However, you don't need to; Apollo CLI provides this functionality with a one liner!

```bash
yarn run apollo schema:download \
  --endpoint http://localhost:3000/your/api/endpoint \
  config/graphql_schema.json
```

Next up, the actually query extraction. What would otherwise be a mammoth task is once again an Apollo CLI one liner!

```bash
yarn run apollo codegen:generate --schema=config/graphql_schema.json \
  --addTypename \
  --queries="app/javascript/**/*.{js,jsx,graphql}" \
  config/graphql_queries.json
```

Now go open `config/graphql_queries.json` and you should find every query and every fragment in your codebase extracted. There's a lot of stuff here, most of which we don't need for our purposes, so we're going to be focussing on just these parts:

```js
{
  // 'operations' contains all the queries found
  "operations": [
    {
      // the operation name of each query is extracted, this makes lookup much easier for us
      "operationName": "HelloWorld",
      // and a version of the query with fragments inlined is also provided so we don't need to
      // stitch that together ourselves
      "sourceWithFragments": "query HelloWorld($id: ID!) {\n  helloWorld(id: $id) {\n    id\n    name\n  }\n}",
    }
  ]
  // ...
}
```

Thats all we need to run our query on any GraphQL client imaginable. For the rest of this tutorial I'll be using the Ruby [graphql-client](https://github.com/github/graphql-client) from Github in a Rails app, but the principles could be applied elsewhere too.

By the way, since query extraction is completely dependent on the contents of your codebase, I recommend adding this file to your `.gitignore` and generating it automatically at application build or deploy time. There's no sense risking this file getting stale, or having it show up in your pull request diffs for no good reason.

### Running Client Queries on the Server

I'll start by setting up `graphql-ruby` on my server, if you already have a functional client, you can skip this part.

First add the client to your server:
```
gem 'graphql-client', '~> 0.13'
```

Now lets create a basic GraphQL Client configured for the endpoint. Its up to you where you put this code, we use `lib/` and require it in a Rails initializer, but do whatever works for you. Here's a minimal version of our client config:

```ruby
require "graphql/client"
require "graphql/client/http"

module MyAPI
  # graphql-client has tooling to save and read your schema itself, but since we have already
  # saved one with Apollo CLI, lets transform that to a format that graphql-client can read
  apollo_cli_output = File.read("config/graphql_schema.json")
  data = { "data" => { "__schema" => JSON.parse(apollo_cli_output) } }
  Schema = GraphQL::Client.load_schema(data)
  HTTP = GraphQL::Client::HTTP.new("https://example.com/graphql") do
    # Customize adapter to your needs, for more info see:
    # https://github.com/github/graphql-client#configuration
  end
  Client = GraphQL::Client.new(schema: Schema, execute: HTTP)
end
```

Now we're going to want some convenience methods to make getting a GraphQL query from our persisted queries file simple. _Here's one I made earlier_:

```ruby
module MyAPI
  class PersistedQueryLookup
    QUERIES_PATH = "config/graphql_queries.json"

    attr_reader :persisted_queries_data

    def initialize
      @persisted_queries_data = load_persisted_queries_data
    end

    def for_operation(name)
      operation = persisted_queries_data["operations"].find { |op| op["operationName"] == name }
      unless operation
        raise RuntimeError, "Could not find persisted query called '#{name}', ensure you ran the Apollo query extractor recently"
      end
      query_with_operation_name = operation["sourceWithFragments"]
      query_with_operation_name.gsub(name, "")
    end

    private

    def load_persisted_queries_data
      unless File.exist?(QUERIES_PATH)
        raise RuntimeError, "Cannot find #{QUERIES_PATH}, ensure you ran the Apollo query extractor"
      end
      data = File.read(QUERIES_PATH)
      JSON.parse(data)
    end
  end
end
```

Now lets add a class method to our `MyAPI` module to cache a singleton of this persisted query lookup class:

```ruby
module MyAPI
  # Append this to the bottom of your MyAPI module...

  # Find a query extracted from the app/javascript build by its operation name
  def self.persisted_query(operation_name)
    persisted_queries.for_operation(operation_name)
  end

  def self.persisted_queries
    @lookup ||= PersistedQueryLookup.new
  end
end
```

Great, now we can easily run a client query from a Rails controller! Since we might be parsing more than one query to our client, we're going to pass into the view a map of operation names to their respective results. This will come in handy when we make a general purpose Apollo cache hydrater.

```ruby
# app/controllers/my_controller.rb
class WorldsController < ApplicationController
  # Parse query extracted from the React / Apollo app, to be used for hydration and server side rendering
  OperationName = "HelloWorld"
  Query = MyAPI::Client.parse(MyAPI.persisted_query(OperationName))

  def index
    result = MyAPI::Client.query(Query, variables: {id: params[:id]})
    @data = { OperationName => result.data }
  end
end
```

### Hydrating the Apollo Client with Prefetched Data

Ok, we've got the query, we've run it on the server, now we need to parse it into the view, and teach apollo how to write it into its internal normalized cache.

In our view, lets parse those results as JSON through to our React component. Here I am using Hypernova, to learn more about that see [my previous post on SSR React in Rails]({{ site.baseurl }}{% post_url 2018-08-04-rails-webpacker-react-ssr %})), but you could just as easily be parsing the prop in via a `window.queryResults` global variable, or another mechanism. The important thing is: when your component mounts, it needs to have this data available to it.

```erb
<% # app/views/worlds/index.html.erb %>

<%=
  render_react_component('HelloWorld', queryResults: @data.to_json)
%>
```

Next, we'll make a general purpose Higher Order Component that we can wrap our HelloWorld component in, that will write these `queryResults` to the Apollo internal cache. It would be nice if this functionality was built into Apollo, but it's not too hard to build ourselves using [Apollo's writeQuery API](https://www.apollographql.com/docs/react/advanced/caching.html#writequery-and-writefragment):


```jsx
// app/javascript/containers/withPreloadedQueries.jsx

import React, { Component } from "React"
import { withApollo } from "react-apollo"

// You'll need to build your map of operation names to gql queries here:
import { query } from "components/HelloWorld"

const queryMap = {
  HelloWorld: query,
}

// Wrap a component with a HOC that will takes a "queryResults" prop of the format:
//  { "MyOperationName" => (JSON data for that operation) }
// ...and populates the Apollo cache with that preloaded data.
export default function withPreloadedQueries(Component) {
  class WithPreloadedQueries extends Component {
    componentWillMount() {
      if (!this.props.queryResults) return
      const queryResults = JSON.parse(this.props.queryResults)
      for (let operationName in queryResults) {
        // Find the matching query to this operation name
        const query = this._lookupQuery(operationName)
        // Write the results to the Apollo Cache
        this.props.client.writeQuery({ query, ...queryResults[operationName] })
      }
    }

    render() {
      const {queryResults, ...rest} = this.props
      return <Component {...rest} />
    }

    _lookupQuery(operationName) {
      const query = queryMap[operationName]
      if (!query) throw new Error(`Missing Query ${operationName}`)
      return query
    }
  }
  return withApollo(WithPreloadedQueries)
}

```

And finally, we need to actually use that HOC by wrapping our HelloWorld component:

```jsx
// app/javascript/components/HelloWorld.jsx

import withPreloadedQueries from "containers/withPreloadedQueries"

// You know... except your component is useful
const HelloWorld = () => {
  return <Stuff/>
}

export default withPreloadedQueries(HelloWorld)
```

And we're done! The HOC has the side effect of synchronously writing the pre-fetched data into the Apollo store. The result is no different from the Apollo client running the query itself. Since its synchronous, your UI should _immediately_ render into its finished state. No loading spinners, no placeholders, just a perfect initial render.

This technique alone allows for really snappy renders of even the most complicated GraphQL component trees, with or without server side rendering. If you aren't worried about SEO, and have repeat visitors that can be expected to have your React code cached, this approach is all you need to have React components render instantly.

Thanks for reading, and a big thanks to the Apollo team for making some of the best frontend software I've ever used.

Matt
