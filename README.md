# Vite SSG

Static-site generation for Vue 3 on Vite.

[![NPM version](https://img.shields.io/npm/v/vite-ssg?color=a1b858)](https://www.npmjs.com/package/vite-ssg)

> ℹ️ **Vite 2 is supported from `v0.2.x`, Vite 1's support is discontinued.**

## Install

> **This library requires Node.js version >= 14**

<pre>
<b>npm i -D vite-ssg</b> <em>vue-router@next @vueuse/head</em>
</pre>

```diff
// package.json
{
  "scripts": {
    "dev": "vite",
-   "build": "vite build"
+   "build": "vite-ssg build"
  }
}
```

```ts
// src/main.ts
import { ViteSSG } from 'vite-ssg'
import App from './App.vue'

// `export const createApp` is required instead of the original `createApp(App).mount('#app')`
export const createApp = ViteSSG(
  // the root component
  App,
  // vue-router options
  { routes },
  // function to have custom setups
  ({ app, router, routes, isClient, initialState }) => {
    // install plugins etc.
  }
)
```

### Single Page SSG

To have SSG for the index page only (without `vue-router`), import from `vite-ssg/single-page` instead, and you only need to install `npm i -D vite-ssg @vueuse/head`.

```ts
// src/main.ts
import { ViteSSG } from 'vite-ssg/single-page'
import App from './App.vue'

// `export const createApp` is required instead of the original `createApp(App).mount('#app')`
export const createApp = ViteSSG(App)
```

### `<ClientOnly/>`

Component `ClientOnly` is registered globally along with the app creation.

```html
<client-only>
  <your-client-side-components />
</client-only>
```

## Document head

From `v0.4.0`, we ship [`@vueuse/head`](https://github.com/vueuse/head) to manage the document head out-of-box. You can directly use it in your pages/components, for example:

```html
<template>
  <button @click="count++">Click</button>
</template>

<script setup>
import { useHead } from '@vueuse/head'

useHead({
  title: 'Website Title',
  meta: [
    {
      name: `description`,
      content: `Website description`,
    },
  ],
})
</script>
```

That's all, no configuration is needed. Vite SSG will handle the server-side rendering and merging automatically.

Refer to [`@vueuse/head`'s docs](https://github.com/vueuse/head) for more usage about `useHead`.

## Critical CSS

Vite SSG has built-in support for generating [Critical CSS](https://web.dev/extract-critical-css/) inlined in the HTML via the [`critters`](https://github.com/GoogleChromeLabs/critters) package. Install it via:

```bash
npm i -D critters
```

Critical CSS generation will be enabled automatically for you.

## Initial State

The initial state comprises data that is serialized to your server-side generated HTML that is hydrated in
the browser when accessed. This data can be data fetched from a CDN, an API, etc, and is typically needed
as soon as the application starts or is accessed for the first time.

The main advantage of setting the application's initial state is that the statically generated pages do not
need to fetch the data again as the data is fetched during build time and serialized into the page's HTML.

The initial state is a plain JavaScript object that can be set during SSR, i.e., when statically generating
the pages, like this:

```ts
// src/main.ts

// ...

export const createApp = ViteSSG(
  App,
  { routes },
  ({ app, router, routes, isClient, initialState }) => {
    // ...

    if (import.meta.env.SSR) {
      // Set initial state during server side
      initialState.data = { cats: 2, dogs: 3 }
    } else {
      // Restore or read the initial state on the client side in the browser
      console.log(initialState.data) // => { cats: 2, dogs: 3 }
    }

    // ...
  }
)
```

Typically, you will use this with an application store, such as
[Vuex](https://vuex.vuejs.org/) or [Pinia](https://pinia.esm.dev/).
For examples, see below:

<details><summary>When using Pinia</summary>
<p>
Following [Pinia's guide](https://pinia.esm.dev/ssr), you will to adapt your `main.{ts,js}` file to look
like this:

```ts
// main.ts
import { ViteSSG } from 'vite-ssg'
import { createPinia } from 'pinia'
import routes from 'virtual:generated-pages'
// use any store you configured that you need data from on start-up
import { useRootStore } from './store/root'
import App from './App.vue'

export const createApp = ViteSSG(
  App,
  { routes },
  ({ app, router, initialState }) => {
    const pinia = createPinia()
    app.use(pinia)

    if (import.meta.env.SSR) {
      initialState.pinia = pinia.state.value
    } else {
      pinia.state.value = initialState.pinia || {}
    }

    router.beforeEach((to, from, next) => {
      const store = useRootStore(pinia)
      if (!store.ready)
        // perform the (user-implemented) store action to fill the store's state
        store.initialize()
      next()
    })
  },
)
```
</p></details>

<details><summary>When using Vuex</summary>
<p>

```ts
// main.ts
import { ViteSSG } from 'vite-ssg'
import routes from 'virtual:generated-pages'
import { createStore } from 'vuex'
import App from './App.vue'

// Normally, you should definitely put this in a separate file
// in order to be able to use it everywhere
const store = createStore({
  // ...
})

export const createApp = ViteSSG(
  App,
  { routes },
  ({ app, router, initialState }) => {
    app.use(store)

    if (import.meta.env.SSR) {
      initialState.store = store.state
    } else {
      store.replaceState(initialState.store)
    }

    router.beforeEach((to, from, next) => {
      // perform the (user-implemented) store action to fill the store's state
      if (!store.getters.ready)
        store.dispatch('initialize')

      next()
    })
  },
)
```
</p></details>

For the example of how to use a store with an initial state in a single page app,
see [the single page example](./examples/single-page/src/main.ts).

### State Serialization

Per default, the state is deserialized and serialized by using `JSON.stringify` and `JSON.parse`.
If this approach works for you, you should definitely stick to it as it yields far better
performance.

You may use the option `transformState` in the `ViteSSGClientOptions` as displayed below.
A valid approach besides `JSON.stringify` and `JSON.parse` is
[`@nuxt/devalue`](https://github.com/nuxt-contrib/devalue) (which is used by Nuxt.js):

```ts
import devalue from '@nuxt/devalue'
import { ViteSSG } from 'vite-ssg'
// ...
import App from './App.vue'

export const createApp = ViteSSG(
  App,
  { routes },
  ({ app, router, initialState }) => {
    // ...
  },
  {
    transformState(state) {
      return import.meta.env.SSR ? devalue(state) : state
    },
  },
)
```

**A minor remark when using `@nuxt/devalue`:** In case, you are getting an error because of a `require`
within the package `@nuxt/devalue`, you have to add the following piece of config to your Vite config:

```ts
// vite.config.ts
//...

export default defineConfig({
  resolve: {
    alias: {
      '@nuxt/devalue': '@nuxt/devalue/dist/devalue.js',
    },
  },
  // ...
})
```

### Async Components

Some applications may make use of Vue features that cause components to render asynchronously (e.g. [`suspense`](https://v3.vuejs.org/guide/migration/suspense.html)). When these features are used in ways that can influence `initialState`, the `onSSRAppRendered` may be used in order to ensure that all async operations have finished as part of the initial application render:

```ts
const { app, router, initialState, isClient, onSSRAppRendered } = ctx;

const pinia = createPinia()
app.use(pinia)

if (isClient) {
  pinia.state.value = (initialState.pinia) || {}
} else {
  onSSRAppRendered(() => {
    initialState.pinia = pinia.state.value
  })
}
```

## Configuration

You can pass options to Vite SSG in the `ssgOptions` field of your `vite.config.js`

```js
// vite.config.js

export default {
  plugins: [ /*...*/ ],
  ssgOptions: {
    script: 'async'
  }
}
```

See [src/types.ts](./src/types.ts). for more options available.

### Custom Routes to Render

You can use the `includedRoutes` hook to exclude/include route paths to render, or even provide some complete custom ones.

```js
// vite.config.js

export default {
  plugins: [ /*...*/ ],
  ssgOptions: {
    includedRoutes(paths, routes) {
      // exclude all the route paths that contains 'foo'
      return paths.filter(i => !i.includes('foo'))
    }
  }
}
```
```js
// vite.config.js

export default {
  plugins: [ /*...*/ ],
  ssgOptions: {
    includedRoutes(paths, routes) {
      // use original route records
      return routes.flatMap(route => {
        return route.name === 'Blog'
          ? myBlogSlugs.map(slug => `/blog/${slug}`)
          : route.path
      })
    }
  }
}
```

Alternatively, you may export the `includedRoutes` hook from your server entry file. This will be necessary if fetching your routes requires the use of environment variables managed by Vite. 

```ts
// main.ts

import { ViteSSG } from 'vite-ssg'
import App from './App.vue'

export const createApp = ViteSSG(
  App,
  { routes },
  ({ app, router, initialState }) => {
    // ...
  },
)
export async function includedRoutes(paths, routes) {
  // Sensitive key is managed by Vite - this would not be available inside 
  // vite.config.js as it runs before the environment has been populated.
  const apiClient = new MyApiClient(import.meta.env.MY_API_KEY) 

  return Promise.all(
    routes.flatMap(async route => {
      return route.name === 'Blog'
        ? (await apiClient.fetchBlogSlugs()).map(slug => `/blog/${slug}`)
        : route.path
    })
  )
}
```

## Comparsion

### Use [Vitepress](https://github.com/vuejs/vitepress) when you want:

- Zero config, out-of-box
- Single-purpose documentation site
- Lightweight ([No double payload](https://twitter.com/youyuxi/status/1274834280091389955))

### Use Vite SSG when you want

- More controls on the build process and tooling
- The flexible plugin systems
- Multi-purpose application with some SSG to improve SEO and loading speed

Cons:

- Double payload

## Example

See [Vitesse](https://github.com/antfu/vitesse)

## Thanks to the prior work

- [vitepress](https://github.com/vuejs/vitepress/tree/master/src/node/build)
- [vue3-vite-ssr-example](https://github.com/tbgse/vue3-vite-ssr-example)
- [vite-ssr](https://github.com/frandiox/vite-ssr)

## License

MIT License © 2020-PRESENT [Anthony Fu](https://github.com/antfu)
