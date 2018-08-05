---
layout: post
title: "Rails + React Server Side Rendering, with Webpacker + Hypernova"
date: 2018-08-04 17:32
comments: true
categories: ruby rails react hypernova ssr "server side rendering"
---


I recently discovered [Hypernova](https://github.com/airbnb/hypernova-ruby), a wonderful tool from Airbnb, making the unimaginable possible: server side rendered React components, in a Rails app, with *no Node server required in development, or even production*.

In the post, I'll walk you through how to make this a reality in four steps:

1. Installing [Webpacker](https://github.com/rails/webpacker) 3.4.1, in a Rails 4.2+ app
2. Installing [Hypernova Ruby Client](https://github.com/airbnb/hypernova-ruby), initially just _client side_ rendering React components
3. Configuring Webpacker + [Hypernova _Server_](https://github.com/airbnb/hypernova), to support rendering those same components _server side_
4. Adding health checks, so we know things are working in production

But first, let me explain what I mean by "no Node server required"... After all, if you want to do server side rendering, you need to run a Node process on your server, so how can you not have a dependency on Node? Well, the architecture of [Hypernova](https://github.com/airbnb/hypernova-ruby) is such that during render, if a connection cannot be quickly established with the Node process, or the process errors during render, it falls back to the usual client side rendering approach. Coming from a Rails shop that over the years has dipped its toes further and further into JS heavy frontend, this is a great comfort to me. I am not a Node guy, I don't have anything against Node, but I definitely don't have expertise in it. So if I can have the benefits of SSR, without adding points of failure, and without massive increase in app complexity, thats a dream come true! Its also great for development; as another engineer, you can come into this project, edit a SSR React component, and check it in your browser, all just from a `rails server` command.

<!-- more -->

I should note, Hypernova has one pretty important constraint: it is synchronous. What this means in practice is **you must pre-fetch critical data in Ruby land**. If you don't do this, at best you'll be server side rendering a loading screen. This is a best practice anyway, since it means even _without_ SSR your component is hydrated with data, and can immediately render meaningful content without loading screens etc. How to achieve this is outside the scope of this post, but for those using GraphQL in their React code, I will write up a post detailing how I handle this in the future.


<small>
_This post is adapted from [another splendid one](https://qiita.com/KeitaMoromizato/items/ea9cf6e787d851ed61e6) by [Keita Moromizato](https://qiita.com/KeitaMoromizato), updated with the (current) latest version of Webpacker, and some deviations for my needs. Thanks Keita! We've never met, and I don't speak Japanese, but I'm standing on your shoulders here_ 😄
</small>

### Step 1: Webpacker

This part is pretty easy, we're just going to follow the installation instructions provided by [Webpacker themselves](https://github.com/rails/webpacker/tree/3-x-stable#installation):

Add to your Rails Gemfile:
```ruby
gem 'webpacker', '~> 3.5'
```

Now run:

```bash
bundle
bundle exec rails webpacker:install webpacker:install:react
```

We're going to make a pack whos exports will be the React component(s) we intend to Server Side Render. This is because in step 3 we will find we need to build that pack twice, once for the client, and once for the server.

```bash
touch app/javascript/packs/react-hypernova.js
```

And finally, ensure the layout you intend to render React components into requires your generated application pack:
```erb
<%= javascript_pack_tag 'react-hypernova' %>
<%= stylesheet_pack_tag 'react-hypernova' %>
```

Now, we're ready to render components. Webpacker hooks into the asset pipeline, compiling your packs on refresh without any extra config. Its _possible_ to run the dev server via `bin/webpack-dev-server`, which has the benefit of supporting React Hot Module Reloading, but that too is for another post.

### Step 2: Hypernova Client

Again, we're following the [getting started instructions](https://github.com/airbnb/hypernova-ruby#getting-started), starting with a gem addition:

```ruby
gem 'hypernova'
```

Now to configure the initializer. This doesn't matter much yet, as we aren't starting the Hypernova _server_ till step 3:
```ruby
# config/initializers/hypernova.rb
Hypernova.configure do |config|
  config.host = "localhost"
  config.port = "3030"
end
```

Next, we need to instrument any controller that will be rendering Hypernova components:

```ruby
class MyController < ApplicationController
  around_action :hypernova_render_support
end
```

Oh, and I guess I'll need a contrived component for rendering, so here's a nice simple [React functional component](https://reactjs.org/docs/components-and-props.html#functional-and-class-components) for us to pretend we would ever want to see:


```jsx
// app/javascript/components/HelloWorld.jsx
import React from "react"

export default function HelloWorld({name, color, shape}) {
  return (
    <div>
      Hi {name}, your favourite color is {color} and shape is {shape}
    </div>
  )
}
```

Now lets render the component! This won't work yet as we haven't told Hypernova how to find the component, but I think it's easier to understand the component registration system when you start from the outside. The syntax Hypernova client provides is real simple, a unique identifier for your component, and a props hash that the client will serialize + de-serialize, parsing into your component as regular props:

```erb
<%=
  render_react_component(
    'HelloWorld',
    name: 'Person',
    color: 'Blue',
    shape: 'Triangle'
  )
%>
```

But as I say, we need to teach Hypernova where to get our component when it receives "HelloWorld". Your pack must call `renderReact()` with the name you wish to use to render your component(s), and the component that will render. Later we'll find it must also _export_ the component for server side rendering, so we'll go ahead and do that right now:

```jsx
// app/javascript/packs/react-hypernova.js
import React from "react"
import { renderReact } from "hypernova-react"

import HelloWorldComponent from "../components/HelloWorld"

export const HelloWorld = renderReact("HelloWorld", HelloWorldComponent)
```

Right now you should be able to load your template, and find your component rendering just fine, even though we haven't set up the server yet!

### Step 3: Server Side Rendering

If you view source on your existing work, you should find that your component is a single `div`, with some data attributes, but we want to replace that with the whole component's DOM, pre-rendered server side, so lets get to it.

First up, lets add the [Hypernova server](https://github.com/airbnb/hypernova) and webpack-merge to our JS dependencies, as we'll need both later:

```bash
yarn add hypernova webpack-merge
```

Next, we'll need to do some damage to our webpacker config to build a version that Node can handle. This will ensure that dependencies that rely on browser APIs and don't crash the server, at least in most cases. I had to do this for instance for the [Mapbox GL](https://github.com/mapbox/mapbox-gl-js) package, which uses WebGL APIs, but knows to short circuit if it detects it is not in a browser environment.

**[Note] Webpacker and Webpack are on the verge of some major changes, so its quite likely that when you read this, its outdated. Sorry, thats JS for you :(**

We're going to add a new webpack config to `config/webpack`, based on our shared `environment` config, but targeting node, and outputting a file with a predictable path (no fingerprinting) so we can fetch it server side without issue:

```js
// config/webpack/server.js
const environment = require("./environment")
const merge = require("webpack-merge")

// React Server Side Rendering webpacker config
// Builds a Node compatible file that Hypernova can load, never served to the client.

const serverEnvironment = merge(environment.toWebpackConfig(), {
  target: "node",
  entry: "./app/javascript/packs/react-hypernova.js",
  output: {
    filename: "server.js",
    path: environment.config.output.path,
    libraryTarget: "commonjs",
  },
})

// This removes the Manifest from the Server.
// Manifest overwrites the _real_ client manifest, required by Rails.
serverEnvironment.plugins = serverEnvironment.plugins
  .filter(plugin => plugin.constructor.name !== "ManifestPlugin")

module.exports = serverEnvironment
```

And we're going to add that `server.js` config to our development and production builds, by replacing the `module.exports` with an _array_, which webpack supports:

```js
// config/webpack/development.js
process.env.NODE_ENV = process.env.NODE_ENV || "development"

const environment = require("./environment")
const serverConfig = require("./server")

module.exports = [environment.toWebpackConfig(), serverConfig]
```

```js
// config/webpack/production.js
process.env.NODE_ENV = process.env.NODE_ENV || "production"

const environment = require("./environment")
const serverConfig = require("./server")

module.exports = [environment.toWebpackConfig(), serverConfig]
```

Now that we have a file that Hypernova server can read, we need to create a script to run the server, and provide it the necessary logic to resolve a component name to a React component. Recall that we configured Rails to look for Hypernova on port 3030, so we'll copy that here.

```js
// script/hypernova.js
const hypernova = require("hypernova/server")

const env = process.env.NODE_ENV || "development"
const devMode = env === "development"

hypernova({
  // Enabling devMode is important because it renders server errors to the
  // DOM, allowing you to know when SSR is failing, and why.
  devMode,
  port: process.env.HYPERNOVA_PORT || 3030,
  getComponent: name => {
    // Allow iteration in dev, because require is cached otherwise
    if (devMode) {
      delete require.cache[require.resolve("../public/packs/server.js")]
    }

    let componentMap = require("../public/packs/server.js")

    if (componentMap[name]) return componentMap[name]
    throw new Error(
      `Could not find component named ${name} in packs/react-hypernova.js, ensure you exported it with the name ${name}`
    )
  },
})
```

Now, when you run your server with `node script/hypernova.js`, and refresh your browser, you should find that your page source includes your pre-rendered component!

When it comes to production, you'll need a way to ensure this script is called with the appropriate RAILS_ENV and NODE_ENV environment variables, and is run on startup. A simple < init system of choice > script should do the trick.


### 4. Production Health Checks

We could stop here, but there's one more piece I recommend adding. The downside of Hypernova's resiliency is that an engineer could break SSR, and no one would know, because your site would still work fine. To counter that, lets use Hypernova client's plugin system to let ourselves how things are working:

```ruby
# Append this to:
# config/initializers/hypernova.rb

class HypernovaRequestMonitoring
  def prepare_request(current_request, original_request)
    # This is a good place to increment internal counters to know that a request happened
    current_request
  end

  def after_response(current_response, original_response)
    # This is a good place to increment internal counters to know that a request was successful
    current_response
  end

  def on_error(error, *args)
    # This is a good place to send SSR render errors to your error tracking system, or log it
  end
end

Hypernova.add_plugin!(HypernovaRequestMonitoring.new)
```

Personally we use this to instrument with some Datadog dogstatsd metrics, log errors in development, and report errors to Sentry in production.


### Conclusion

Alright! You _should_ now have a Rails + React + Webpacker + Hypernova setup, with SSR working a treat. If you don't, its probably because I extracted this 101 tutorial from a more complicated project, and missed a part. Please leave a comment, and I will do my best to correct any mistakes quickly. If it's not that, then you might like me be the victim of constant change in Webpack + Webpacker. At time or writing, Webpacker 4.0 is in pre-release, hopefully it doesn't change too much!

Thanks for reading,

Matt
