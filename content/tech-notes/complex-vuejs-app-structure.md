+++
date = 2017-01-25T03:23:55+05:30
draft = false
title = "Complex Vue.js App Structure"

aliases = ["/posts/complex-vuejs-app-structure"]

[taxonomies]
tags = ["vue", "javascript", "web", "frontend"]
+++

I have always liked it whenever a framework provides it's own generators and/or boilerplates. It's what I liked about [Ruby on Rails](http://rubyonrails.org) the most after the [MVC](https://en.wikipedia.org/wiki/Model–view–controller) concept. Developers can understand a lot about the framework and the way it's intended to be used from the official boilerplates. I am sad that the recent javascript frameworks has left this important job of generating code to the community and tools like yeoman instead of having something official.

When I started looking into [Vue.js][], I was happy to discover that they have a small generator tool called **vue-cli** which can be used with official boilerplates provided at [vuejs-templates](https://github.com/vuejs-templates). Unfortunately, the happiness didn't last long because the boilerplates are simple applications meant to get you started with [Vue.js][]. They don't deal with all the other necessary packages such as [vuex][], [vue-router][], etc.. which are needed for a complex [Vue.js][] application. *(__EDIT__: At the time of writing this post, vue-router wasn't present in the boilerplate)*. They did allow the tool to use third party boilerplates which resulted in me creating [spoiler](https://github.com/pksunkara/spoiler) built upon the official webpack boilerplate.

I would like to describe below about what I think should be the directory and file structure of a complex [Vue.js][] application and it's conventions.

## Overview

The following example assumes that you will be using webpack build config from [vuejs-templates/webpack](https://github.com/vuejs-templates/webpack) boilerplate.

Let's also assume that you want to have a page named **Hello**, it will need a [vue.js][] component named **Hello** and a [vuex][] module named **Hello**.

```bash
.
├─ src
│  ├─ assets               # module assets (processed by webpack)
│  │  └─ ...
│  ├─ components
│  │  └─ Hello.vue         # Hello component
│  │  └─ ...
│  ├─ store
│  │  └─ Hello.js          # Hello module
│  │  └─ index.js          # assemble vuex store
│  │  └─ ...
│  ├─ router
│  │  └─ index.js          # app route configuration
│  │  └─ ...
│  ├─ App.vue              # main app component
│  └─ main.js              # app entry file
├─ public                  # static assets (directly copied)
│  └─ ...
├─ index.html              # index.html template
└─ package.json            # build scripts and dependencies
```

#### index.html

This is the main HTML template for your application. You can link your static assets inside the **head** tag while your processed assets will be auto injected in **body**.

```html
<!DOCTYPE html>
<html>
  <head>
    <link rel="shortcut icon" type="image/png" href="/assets/images/favicon.png"/>
  </head>

  <body>
    <div id="app"></div>
  </body>
</html>
```

#### src/router/index.js

This is the file which initiates [vue-router][] with the given components.

```js
import Vue from 'vue';
import VueRouter from 'vue-router';
import Hello from '@/components/Hello';

Vue.use(VueRouter);

const routes = [
  { path: '/', name: 'Hello', component: Hello },
];

/* eslint-disable no-new */
export default new VueRouter({
  routes,
  mode: 'history',
});
```

#### src/store/index.js

This is the file which initiates [vuex][] store with the given modules.

```js
import Vue from 'vue';
import Vuex from 'vuex';
import Hello from '@/store/Hello';

Vue.use(Vuex);

/* eslint-disable no-new */
const store = new Vuex.Store({
  modules: {
    Hello,
  },
});

export default store;
```

#### src/App.vue

This is the application's main [Vue.js][] component which is basically just a wrapper for [vue-router][].

```vue
<template>
  <div id="app">
    <router-view></router-view>
  </div>
</template>

<script>
export default {};
</script>
```

#### src/main.js

This is the application entry file where you initiate your [Vue.js][] application with a router, store and the main **App.vue** component.

```js
import Vue from 'vue';
import router from '@/router';
import store from '@/store';
import App from '@/App';

/* eslint-disable no-new */
new Vue({
  el: '#app',
  router,
  store,
  render: h => h(App),
});
```

#### src/store/Hello.js

This file represents a sample [vuex][] module.

```js
export default {
  namespaced: true,
  state: {
    message: 'Hello Vue!',
  },
};
```

#### src/components/Hello.vue

This file represents a sample [Vue.js][] component which will be used by the [vue-router][]. Please note that **Hello** module's state is being used in here.

```vue
<template>
  <div class="hello">
    <h1>{{ message }}</h1>
  </div>
</template>

<script>
import { mapState } from 'vuex';

export default {
  data() {
    return {};
  },
  computed: mapState({
    message: state => state.Hello.message,
  }),
};
</script>
```

## Questions

#### Why is router a directory instead of a single file?

When applications grow, the number of routes grow. They might contain **beforeEnter** hooks and so on and sometimes you might want to put all that in a separate file, or files. With a subdirectory you can keep this all in one place without bloating one router.js file.

#### Why are there no root actions and mutations in vuex?

In large applications, almost everything should be as independent of each other for better maintainability and readability of the codebase. Having root actions and mutations will hamper that and thus is not recommended.

## Final Comments

I wish the official webpack template supports [vue-router][], [vuex][], etc.. and hopefully guide lot of developers to the correct way of using [Vue.js][]. I also wish it contains some subcommands which can be used when creating a new component or module so that the official conventions are encouraged.

[Vue.js]: https://vuejs.org
[vuex]: https://vuex.vuejs.org/en/
[vue-router]: https://router.vuejs.org/en/
