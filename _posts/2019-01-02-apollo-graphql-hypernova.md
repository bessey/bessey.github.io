---
layout: post
title: "Server Side Rendering for React + Apollo GraphQL Client"
date: 2019-01-02 13:01
comments: true
categories: ruby rails react apollo graphql
---

In a previous post I talked about how I set up [server side rendering of React components in Rails with Hypernova]({{ site.baseurl }}{% post_url 2018-08-04-rails-webpacker-react-ssr %}). I went on to [build a complex Ruby based GraphQL data pre-fetcher]({{ site.baseurl }}{% post_url 2018-08-05-apollo-graphql-prefetching %}) because I could not work out how to do asynchronous pre-fetching of data in Hypernova alone. Well, it turns out though unclearly documented, its actually quite easy to do async pre-render work in Hypernova. That means we don't need to do the crazy things I was doing in my previous post, and can instead pull off my dream: **server side rendered React, served by the back-end language / framework of your choosing.**

I'm going to assume you've already [set up Hypernova with Rails]({{ site.baseurl }}{% post_url 2018-08-04-rails-webpacker-react-ssr %}) (or the back-end of your choosing). In this post, I will cover:
1. Understanding how Hypernova's renderer works
2. Extending it to support pre-fetching GraphQL data with Apollo Client
3. Extracting this GraphQL data for rehydration on the client's Apollo Client
4. Server side rendering your component with this pre-fetched data included

Throughout this I will use snippets of code from an example repo I put together, occasionally modified for brevity. If you prefer to see a complete example up front, see [my example repo on GitHub](https://github.com/bessey/hypernova_apollo_rails).

### 1. Understanding the Hypernova Renderer

If you followed my previous post, you should have a file something like this where you register your React components with the Hypernova renderer.

```jsx
// app/javascript/packs/react-hypernova.js
import React from "react"
import { renderReact } from "hypernova-react"

import HelloWorldComponent from "../components/HelloWorld"

export const HelloWorld = renderReact("HelloWorld", HelloWorldComponent)
```

In order to customize `renderReact` to our needs, we first need to understand it. We can dive into the [hypernova-react source code](https://github.com/airbnb/hypernova-react/blob/82b92bafcc090b8a046062590e310b8fef2be029/src/index.js) to learn  what exactly `renderReact` does. Here's the source code for posterity:
```jsx
// hypernova-react/src/index.js
export const renderReact = (name, component) => hypernova({
  server() {
    return (props) => {
      const contents = ReactDOMServer.renderToString(React.createElement(component, props));
      return serialize(name, contents, props);
    };
  },

  client() {
    const payloads = load(name);

    if (payloads) {
      payloads.forEach((payload) => {
        const { node, data } = payload;
        const element = React.createElement(component, data);

        if (ReactDOM.hydrate) {
          ReactDOM.hydrate(element, node);
        } else {
          ReactDOM.render(element, node);
        }
      });
    }

    return component;
  },
});
```
So to summarize the above implementation, `hypernova-react` is passing `hypernova()` two functions. First, one for rendering a given `component` on the _server_. Then, on the _client_, one for finding that rendered HTML in the DOM and rehydrating it as a live React component.

But what is the `hypernova()` call _doing_ with these functions? If we go look [in _it's_ source code](https://github.com/airbnb/hypernova/blob/fdf6b8c06d8bcc9520a9e532735a3fce4c48c8c3/src/index.js#L85) we can see the answer is not much at all!

```js
// hypernova/src/index.js
export default function hypernova(runner) {
  return typeof window === 'undefined'
    ? runner.server()
    : runner.client();
}
```

It turns out the Hypernova call itself is just a helper function to call the appropriate client / server function we passed in based on environment. No fancy business going on at all! This means we can now be confident in knowing the only place that needs to change is the `server()` implementation passed into the `renderReact` definition.

### 2. Pre-fetching Apollo GraphQL Data on the Server

We now know what needs changing, `server()` needs to pre-fetch our data requirements before calling `ReactDOMServer.renderToString`. But how can we achieve that? Well fortunately the excellent developers of Apollo have our backs. The `react-apollo` library's [getDataFromTree](https://www.apollographql.com/docs/react/features/server-side-rendering.html#getDataFromTree) provides a simple way to build a Promise that will only resolve when all data has been loaded for a given React component tree. Its so unbelievably simple that in fact that it's practically magic, here's an example of its usage:

```jsx
import { getDataFromTree } from "react-apollo"

// 1. Build the JSX tree
const App = <MyComponentThatFetchesData/>
// 2. Extract the data requirements for the given tree with Apollo
getDataFromTree(App).then(() => {
  // 3. Now render the tree, knowing that the data has been fetched!
  return ReactDOM.renderToString(App);
})
```

### 3. Passing the Pre-fetched GraphQL Data For Rehydration on the Client

Unfortunately, it's not enough to render the component with its data loaded. Doing so will initially render your component as you desire, but the second React starts, the components will revert to its 'loading' state, because the browser's Apollo Client cache is empty! We'll also need to extract the contents of the server's Apollo Client cache, and serialize it in some way that the browser's can read.

Extracting the cache for rehydration is fairly easy:
```js
import ApolloClient from "apollo-boost"
import { InMemoryCache } from "apollo-cache-inmemory"
const client = new ApolloClient()
const data = client.extract()
```
... as is hydrating a new client with that cache data:
```js
const cache = new InMemoryCache({}).restore(data)
const newClient = new ApolloClient({cache: cache})
```

So you will have to ensure your components support passing in an Apollo Client instance through props (for the _server_), but fall back to creating a new instance on the _client_. When creating a new Apollo Client instance, it should hydrate from `window.SOME_GLOBAL`. We'll get to how to get our cached data onto `window.SOME_GLOBAL` shortly. To keep all this functionality composable we can use a Higher Order Component:

```jsx
// app/javascript/containers/withApollo.js
import React from "react";
import { ApolloProvider } from "react-apollo";
import ApolloClient from "apollo-boost";
import { InMemoryCache } from "apollo-cache-inmemory";

export default ComposedComponent => {
  return class WithApollo extends React.Component {
    constructor(props) {
      super(props);
      this.apollo =
        props.apolloClient ||
          new ApolloClient({
            uri: "https://api.graphcms.com/simple/v1/swapi",
            cache: new InMemoryCache({}).restore((window && window.__APOLLO_STATE__) || {})
          });
    }

    render() {
      return (
        <ApolloProvider client={this.apollo}>
          <ComposedComponent {...this.props} />
        </ApolloProvider>
      );
    }
  };
};
```

Now, as for getting the cache into `window.__APOLLO_STATE__`... My devious solution for this is rendering a script tag alongside the component, so we can load our Apollo Client data from JSON, onto `window`, and then remove ourself _before_ React rehydrates. That's important, because in the client's view that script tag should not be in the Component's DOM. Luckily enough thanks to the [document.currentScript](https://developer.mozilla.org/en-US/docs/Web/API/Document/currentScript) API, it is possible for a `<script/>` tag to remove itself from the DOM during execution. The final result is this:

{% raw %}
```jsx
// app/javascript/hypernovaApolloRenderer.jsx
import React, { Fragment } from "react";

function buildApolloClientCacheComponent(Component, clientCache) {
  const ComponentWithCache = props => (
    <Fragment>
      {/*
        Store Apollo Client state, per
        https://www.apollographql.com/docs/react/features/server-side-rendering.html#getDataFromTree
        then delete ourself because React will warn it wasn't expecting a script tag.
        Deletion doesn't work in IE, but the only harm done is generating that React warning.
      */}
      <script
        dangerouslySetInnerHTML={{
          __html: `window.__APOLLO_STATE__=${JSON.stringify(
            clientCache
          ).replace(/</g, "\\u003c")};
          document.currentScript && document.currentScript.remove();
          `
        }}
      />
      <Component {...props} />
    </Fragment>
  );
  return ComponentWithCache;
}
```
{% endraw %}

### 4. Server Side Rendering the Component With Pre-fetched Data

So now all that's left to do is integrate all this into our `server()` implementation, but how? Well, here's the crucial part I missed last time: What isn't mentioned in the documentation for `hypernova-react` or anywhere that I can see in `hypernova` is that `server()` can **return a Promise**. The provided implementation is synchronous, but it does not have to be. Armed with this knowledge, the remaining changes are relatively small:

```jsx
// app/javascript/hypernovaApolloRenderer.jsx
import React, { Fragment } from "react";
import ReactDOM from "react-dom";
import ReactDOMServer from "react-dom/server";
import hypernova, { serialize, load } from "hypernova";
import ApolloClient from "apollo-boost";

import { getDataFromTree } from "react-apollo";
import createApolloClient from "./createApolloClient";

export const renderReactWithApollo = (name, Component) => hypernova({
  server() {
    // 1. We make the function async so we can await in it
    return async props => {
      // 2. With a client instance we have access to, fetch the data from the component
      const client = new ApolloClient({ uri: "https://api.graphcms.com/simple/v1/swapi" });
      await getDataFromTree(<Component {...props} apolloClient={client} />);

      // 3. With our previously built cache side-loader, build a new component with the cache included in JSON
      const ComponentWithCache = buildApolloClientCacheComponent(
        Component,
        client.extract()
      );
      // 4. Render with our prehydrated client, so we don't SSR a Loading screen
      const element = React.createElement(ComponentWithCache, {
        ...props,
        apolloClient: client
      });
      const html = ReactDOMServer.renderToString(element);
      return serialize(name, html, props);
    };
  },
  // client() is unchanged
  // ...
});
```

And with that, you're done, you have everything you need to server side render React components with Apollo data pre-fetched, or so I hope. If you've gotten lost anywhere in this guide, or just want to see the whole lot put together, check out this [complete example repo on my GitHub](https://github.com/bessey/hypernova_apollo_rails).
