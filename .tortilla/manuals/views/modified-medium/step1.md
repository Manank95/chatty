# Step 1: Setup

This is the first blog in a multipart series where we will be building Chatty, a WhatsApp clone, using [React Native](https://facebook.github.io/react-native/) and [Apollo](http://dev.apollodata.com/).

# Overview
Each part of this series will be focused on teaching a core concept of Apollo or React Native. We’ll start from scratch, and by the end of the series, we’ll have a kick-ass group messaging application with real-time updates. Future posts beyond the core series will cover more complex features like push notifications, file uploads, and query optimizations.

Since we are using shiny new tech, this series will be a living document. I will update each post as the tools we use continue to advance. My goal is to use this series as a best practices model for building a complex application using some of the best software available.

With that in mind, if you have any suggestions for making this series better, please leave your feedback!

# The Stack
Chatty will use the following stack:
* Server: Express
* Client: React Native
* Middleware: Apollo (GraphQL)
* Database: SQL (sqlite to start)

This is a pretty awesome stack for building complex real-time native applications.

For those of you who are new to Apollo, I just want to point out some of the coolest built-in features for [Apollo with React](http://dev.apollodata.com/react/):
* Smart query caching (client side state gets updated and cached with each query/mutation)
* Subscriptions (realtime updates pushed by server)
* Optimistic UI (UI that predicts how the server will respond to a request)
* SSR support
* Prefetching

That’s a ton of buzzwords! In the end, what that all really adds up to is our app will be data driven, really fast for users, and get real-time updates as they happen.

# Part 1 Goals
Here’s what we are going to accomplish in this first tutorial:
1. Set up our dev environment
2. Start a basic express server
3. Create our first GraphQL Schema
4. Start a basic React Native client
5. Connect our express server and RN client with Apollo

# Getting started
For this tutorial series, we’re going to start from absolute scratch. My style is to keep everything really simple and refactor as we add complexity.
Let’s start with this basic directory structure:
```
/chatty
  /node_modules
  package.json
  /server
    ... express files
  /client
    /node_modules
    package.json
    ... RN files
```
We will keep our React Native code separate from our server code. This will also keep server dependencies separate from React Native dependencies, which means **we will have 2 `package.json` files**. That may sound weird/bad, but trying to get everything set up with one packager is a huge hassle. It will also save us from a few other issues down the line.

Here’s the terminal code to get us started:
```
# make our directory
mkdir chatty
cd chatty

# start yarn package managing
yarn init

# build some server folders and files
mkdir server
cd server
touch index.js
```
## Setting up the dev environment
We’ll start setting up our dev env with the following features:
1. Server stays running and reloads when we modify code
2. ES6 syntax including import syntax in our server code
3. ESLint with AirBNB presets
```
# from root dir..

# add dev dependencies
yarn global add eslint-cli # eslint is an excellent linter

yarn add --dev babel-cli babel-preset-es2015 babel-preset-stage-2 nodemon eslint babel-eslint
eslint --init  # choose airbnb preset or your preferred setup
```

My `eslintrc.js` file looks like this:

[{]: <helper> (diffStep 1.2 files=".eslintrc.js")

#### Step 1.2: Add eslint, babel, and nodemon

##### Added .eslintrc.js
```diff
@@ -0,0 +1,18 @@
+┊  ┊ 1┊module.exports = {
+┊  ┊ 2┊    "parser": "babel-eslint",
+┊  ┊ 3┊    "extends": "airbnb",
+┊  ┊ 4┊    "plugins": [
+┊  ┊ 5┊        "react",
+┊  ┊ 6┊        "jsx-a11y",
+┊  ┊ 7┊        "import"
+┊  ┊ 8┊    ],
+┊  ┊ 9┊    "rules": {
+┊  ┊10┊        "react/jsx-filename-extension": [1, { "extensions": [".js", ".jsx"] }],
+┊  ┊11┊        "react/require-default-props": [0],
+┊  ┊12┊        "react/no-unused-prop-types": [2, {
+┊  ┊13┊            "skipShapeProps": true
+┊  ┊14┊        }],
+┊  ┊15┊        "react/no-multi-comp": [0],
+┊  ┊16┊        "no-bitwise": [0],
+┊  ┊17┊    },
+┊  ┊18┊};🚫↵
```

[}]: #

Create our start script inside `package.json`:

[{]: <helper> (diffStep 1.3 files="package.json")

#### Step 1.3: Create start script

##### Changed package.json
```diff
@@ -7,6 +7,9 @@
 ┊ 7┊ 7┊  "repository": "https://github.com/srtucker22/chatty.git",
 ┊ 8┊ 8┊  "author": "Simon Tucker <srtucker22@gmail.com>",
 ┊ 9┊ 9┊  "license": "MIT",
+┊  ┊10┊  "scripts": {
+┊  ┊11┊    "start": "nodemon --watch server --watch package.json server/index.js --exec babel-node --presets es2015,stage-2"
+┊  ┊12┊  },
 ┊10┊13┊  "devDependencies": {
 ┊11┊14┊    "babel-cli": "^6.24.1",
 ┊12┊15┊    "babel-eslint": "^8.2.1",
```

[}]: #

## Starting the Express server
Let’s import express in `index.js` using ES6 syntax.
1. `yarn add express`
2. Add the following to `index.js`:

[{]: <helper> (diffStep 1.4 files="index.js")

#### Step 1.4: Add express

##### Changed server&#x2F;index.js
```diff
@@ -0,0 +1,7 @@
+┊ ┊1┊import express from 'express';
+┊ ┊2┊
+┊ ┊3┊const PORT = 8080;
+┊ ┊4┊
+┊ ┊5┊const app = express();
+┊ ┊6┊
+┊ ┊7┊app.listen(PORT, () => console.log(`Server is now running on http://localhost:${PORT}`));
```

[}]: #

Quickly verify our setup works by running `yarn start`.

We have a great starting point. Our start script will transpile ES6 code, spin up our express server, and refresh as we make changes to server code. Nice!

## Adding GraphQL to Express
[GraphQL](http://graphql.org/) in a nutshell is a query language for APIs. It’s a middleware that sits between your server side data and your client. It allows your client to query for exactly what it needs in one single trip and nothing more. You can check out [GraphQL’s homepage](http://graphql.org/) for some sweet visualizations illustrating why GraphQL is so cool.

We’ll start by creating a basic GraphQL Schema. A Schema establishes the data types the client can request and how the client is allowed to request them.

We’ll create a new folder `/server/data` and add a new file `schema.js`:

[{]: <helper> (diffStep 1.5)

#### Step 1.5: Create basic schema

##### Added server&#x2F;data&#x2F;schema.js
```diff
@@ -0,0 +1,10 @@
+┊  ┊ 1┊export const Schema = [
+┊  ┊ 2┊  `type Query {
+┊  ┊ 3┊    testString: String
+┊  ┊ 4┊  }
+┊  ┊ 5┊  schema {
+┊  ┊ 6┊    query: Query
+┊  ┊ 7┊  }`,
+┊  ┊ 8┊];
+┊  ┊ 9┊
+┊  ┊10┊export default Schema;
```

[}]: #

Apollo requires a list of strings written in GraphQL’s language to establish a Schema. This Schema will just be a basic placeholder for now. We will add more advanced and meaningful Schemas in the next tutorial.

We also need our Schema to work with data. A great way to get Schemas up and running is by mocking data. Mocking data also happens to be useful for testing, so it’s good practice to start using mocks with Schemas before attaching real data like a database or 3rd party API.

We’ll add the file `/server/data/mocks.js`:

[{]: <helper> (diffStep 1.6)

#### Step 1.6: Create basic mocks

##### Added server&#x2F;data&#x2F;mocks.js
```diff
@@ -0,0 +1,5 @@
+┊ ┊1┊export const Mocks = {
+┊ ┊2┊  String: () => 'It works!',
+┊ ┊3┊};
+┊ ┊4┊
+┊ ┊5┊export default Mocks;
```

[}]: #

Using the `Mocks` Object, we will be able to convert all Strings returned by GraphQL queries to “It works!”

We want to add a GraphQL endpoint to our server in `server/index.js` so clients can use GraphQL with our server. First we need to add the following dependencies:

```
yarn add body-parser graphql graphql-server-express graphql-tools
```

We’ll rewrite `server/index.js` as follows (explanation below):

[{]: <helper> (diffStep 1.7 files="index.js")

#### Step 1.7: Add graphqlExpress

##### Changed server&#x2F;index.js
```diff
@@ -1,7 +1,35 @@
 ┊ 1┊ 1┊import express from 'express';
+┊  ┊ 2┊import { graphqlExpress, graphiqlExpress } from 'apollo-server-express';
+┊  ┊ 3┊import { makeExecutableSchema, addMockFunctionsToSchema } from 'graphql-tools';
+┊  ┊ 4┊import bodyParser from 'body-parser';
+┊  ┊ 5┊import { createServer } from 'http';
 ┊ 2┊ 6┊
-┊ 3┊  ┊const PORT = 8080;
+┊  ┊ 7┊import { Schema } from './data/schema';
+┊  ┊ 8┊import { Mocks } from './data/mocks';
 ┊ 4┊ 9┊
+┊  ┊10┊const GRAPHQL_PORT = 8080;
 ┊ 5┊11┊const app = express();
 ┊ 6┊12┊
-┊ 7┊  ┊app.listen(PORT, () => console.log(`Server is now running on http://localhost:${PORT}`));
+┊  ┊13┊const executableSchema = makeExecutableSchema({
+┊  ┊14┊  typeDefs: Schema,
+┊  ┊15┊});
+┊  ┊16┊
+┊  ┊17┊addMockFunctionsToSchema({
+┊  ┊18┊  schema: executableSchema,
+┊  ┊19┊  mocks: Mocks,
+┊  ┊20┊  preserveResolvers: true,
+┊  ┊21┊});
+┊  ┊22┊
+┊  ┊23┊// `context` must be an object and can't be undefined when using connectors
+┊  ┊24┊app.use('/graphql', bodyParser.json(), graphqlExpress({
+┊  ┊25┊  schema: executableSchema,
+┊  ┊26┊  context: {}, // at least(!) an empty object
+┊  ┊27┊}));
+┊  ┊28┊
+┊  ┊29┊app.use('/graphiql', graphiqlExpress({
+┊  ┊30┊  endpointURL: '/graphql',
+┊  ┊31┊}));
+┊  ┊32┊
+┊  ┊33┊const graphQLServer = createServer(app);
+┊  ┊34┊
+┊  ┊35┊graphQLServer.listen(GRAPHQL_PORT, () => console.log(`GraphQL Server is now running on http://localhost:${GRAPHQL_PORT}/graphql`));
```

[}]: #

What we’ve done is add Apollo’s `graphqlExpress` and `graphiqlExpress` middleware for the `/graphql` endpoint. The `graphqlExpress` middleware enables clients to retrieve data by querying with our Schema. However, since we don’t have real data yet, we can use `Mocks` to fake the data when our schema is queried by using `addMockFunctionsToSchema`.

We’ve also added a second endpoint `/graphiql`, which uses the `graphiqlExpress` middleware. This middleware connects to our GraphQL endpoint and displays an UI for sending GraphQL queries to our server, called GraphIQL.

Let’s test it all out. Open `http://localhost:8080/graphiql` and you should see the GraphIQL interface. Type in `{testString}` and you should get back the proper response:
![GraphIQL Image](https://s3-us-west-1.amazonaws.com/tortilla/chatty/step1-7.png)

Great! So now we have a server that runs the most basic GraphQL. We could build up our GraphQL backend a bit more, but I’d prefer to connect our server and React Native client before we make our Schema any more complex.

# Starting the React Native client
First we’ll download the dependencies and initialize our React Native app. For the sake of brevity, I’m going to focus on iOS, but all our code should also work with Android.

```
# from root dir...
yarn global add react-native-cli

# initialize RN with name chatty
react-native init chatty

# change name of RN folder to client
mv chatty client

# run the app in simulator
cd client
react-native run-ios # and grab a snack or something cause this might take a while the first run...
```
Running the initialization will create an `index.ios.js` file. In this file is boilerplate code that creates a React component and registers it with `AppRegistry`, which renders the component.

Let’s pull out the `Chatty` component from `index.ios.js` and stick it in its own file. I prefer to organize my files by type rather than feature, but you’re welcome to organize differently if you feel strongly about it.

So I’m going to place the `Chatty` component code into `client/src/app.js` and rename the component `App`.

[{]: <helper> (diffStep 1.9)

#### Step 1.9: Move app code to /src

##### Changed client&#x2F;index.js
```diff
@@ -1,4 +1,4 @@
 ┊1┊1┊import { AppRegistry } from 'react-native';
-┊2┊ ┊import App from './App';
+┊ ┊2┊import App from './src/app';
 ┊3┊3┊
 ┊4┊4┊AppRegistry.registerComponent('chatty', () => App);
```

##### Changed client&#x2F;App.js
```diff
@@ -1,42 +1,10 @@
-┊ 1┊  ┊/**
-┊ 2┊  ┊ * Sample React Native App
-┊ 3┊  ┊ * https://github.com/facebook/react-native
-┊ 4┊  ┊ * @flow
-┊ 5┊  ┊ */
-┊ 6┊  ┊
 ┊ 7┊ 1┊import React, { Component } from 'react';
 ┊ 8┊ 2┊import {
-┊ 9┊  ┊  Platform,
 ┊10┊ 3┊  StyleSheet,
 ┊11┊ 4┊  Text,
-┊12┊  ┊  View
+┊  ┊ 5┊  View,
 ┊13┊ 6┊} from 'react-native';
 ┊14┊ 7┊
-┊15┊  ┊const instructions = Platform.select({
-┊16┊  ┊  ios: 'Press Cmd+R to reload,\n' +
-┊17┊  ┊    'Cmd+D or shake for dev menu',
-┊18┊  ┊  android: 'Double tap R on your keyboard to reload,\n' +
-┊19┊  ┊    'Shake or press menu button for dev menu',
-┊20┊  ┊});
-┊21┊  ┊
-┊22┊  ┊export default class App extends Component {
-┊23┊  ┊  render() {
-┊24┊  ┊    return (
-┊25┊  ┊      <View style={styles.container}>
-┊26┊  ┊        <Text style={styles.welcome}>
-┊27┊  ┊          Welcome to React Native!
-┊28┊  ┊        </Text>
-┊29┊  ┊        <Text style={styles.instructions}>
-┊30┊  ┊          To get started, edit App.js
-┊31┊  ┊        </Text>
-┊32┊  ┊        <Text style={styles.instructions}>
-┊33┊  ┊          {instructions}
-┊34┊  ┊        </Text>
-┊35┊  ┊      </View>
-┊36┊  ┊    );
-┊37┊  ┊  }
-┊38┊  ┊}
-┊39┊  ┊
 ┊40┊ 8┊const styles = StyleSheet.create({
 ┊41┊ 9┊  container: {
 ┊42┊10┊    flex: 1,
```
```diff
@@ -55,3 +23,22 @@
 ┊55┊23┊    marginBottom: 5,
 ┊56┊24┊  },
 ┊57┊25┊});
+┊  ┊26┊
+┊  ┊27┊export default class App extends Component {
+┊  ┊28┊  render() {
+┊  ┊29┊    return (
+┊  ┊30┊      <View style={styles.container}>
+┊  ┊31┊        <Text style={styles.welcome}>
+┊  ┊32┊          Welcome to React Native!
+┊  ┊33┊        </Text>
+┊  ┊34┊        <Text style={styles.instructions}>
+┊  ┊35┊          To get started, edit index.ios.js
+┊  ┊36┊        </Text>
+┊  ┊37┊        <Text style={styles.instructions}>
+┊  ┊38┊          Press Cmd+R to reload,{'\n'}
+┊  ┊39┊          Cmd+D or shake for dev menu
+┊  ┊40┊        </Text>
+┊  ┊41┊      </View>
+┊  ┊42┊    );
+┊  ┊43┊  }
+┊  ┊44┊}
```

[}]: #

## Adding Apollo to React Native

We’re going to modify `app.component.js` to use [React-Apollo](http://dev.apollodata.com/react/) and [Redux](http://redux.js.org/). While Apollo can be used sans Redux, the developer experience for React Native is much sweeter with Redux for monitoring our app's state, as you'll soon see.

We need to add a bunch of Apollo packages and a couple Redux ones:
```
# **make sure we're adding all react native and client related packages to package.json in the client folder!!!**
cd client

yarn add apollo-cache-redux apollo-client apollo-link apollo-link-http apollo-link-redux graphql graphql-tag react-apollo react-redux redux redux-devtools-extension
```
We need to do the following:
1. Create a Redux store
2. Create an Apollo client
3. Connect our Apollo client to our GraphQL endpoint via the `apollo-link-http`
4. Connect Redux to our Apollo workflow via `apollo-link-redux`. This will let us track Apollo events as Redux actions!
5. Set our Apollo client's data store (cache) to Redux via `apollo-cache-redux`

We can also swap out `compose` for `composeWithDevTools`, which will let us observe our Redux state remotely via [React Native Debugger](https://github.com/jhen0409/react-native-debugger).

[{]: <helper> (diffStep "1.10")

#### Step 1.10: Add ApolloClient

##### Changed client&#x2F;package.json
```diff
@@ -7,8 +7,19 @@
 ┊ 7┊ 7┊		"test": "jest"
 ┊ 8┊ 8┊	},
 ┊ 9┊ 9┊	"dependencies": {
+┊  ┊10┊		"apollo-cache-redux": "^0.1.0-alpha.7",
+┊  ┊11┊		"apollo-client": "^2.2.3",
+┊  ┊12┊		"apollo-link": "^1.1.0",
+┊  ┊13┊		"apollo-link-http": "^1.3.3",
+┊  ┊14┊		"apollo-link-redux": "^0.2.1",
+┊  ┊15┊		"graphql": "^0.12.3",
+┊  ┊16┊		"graphql-tag": "^2.4.2",
 ┊10┊17┊		"react": "16.2.0",
-┊11┊  ┊		"react-native": "0.52.0"
+┊  ┊18┊		"react-apollo": "^2.0.4",
+┊  ┊19┊		"react-native": "0.52.0",
+┊  ┊20┊		"react-redux": "^5.0.5",
+┊  ┊21┊		"redux": "^3.7.2",
+┊  ┊22┊		"redux-devtools-extension": "^2.13.2"
 ┊12┊23┊	},
 ┊13┊24┊	"devDependencies": {
 ┊14┊25┊		"babel-jest": "20.0.3",
```

##### Changed client&#x2F;src&#x2F;app.js
```diff
@@ -5,6 +5,42 @@
 ┊ 5┊ 5┊  View,
 ┊ 6┊ 6┊} from 'react-native';
 ┊ 7┊ 7┊
+┊  ┊ 8┊import { ApolloClient } from 'apollo-client';
+┊  ┊ 9┊import { ApolloLink } from 'apollo-link';
+┊  ┊10┊import { ApolloProvider } from 'react-apollo';
+┊  ┊11┊import { composeWithDevTools } from 'redux-devtools-extension';
+┊  ┊12┊import { createHttpLink } from 'apollo-link-http';
+┊  ┊13┊import { createStore, combineReducers } from 'redux';
+┊  ┊14┊import { Provider } from 'react-redux';
+┊  ┊15┊import { ReduxCache, apolloReducer } from 'apollo-cache-redux';
+┊  ┊16┊import ReduxLink from 'apollo-link-redux';
+┊  ┊17┊
+┊  ┊18┊const URL = 'localhost:8080'; // set your comp's url here
+┊  ┊19┊
+┊  ┊20┊const store = createStore(
+┊  ┊21┊  combineReducers({
+┊  ┊22┊    apollo: apolloReducer,
+┊  ┊23┊  }),
+┊  ┊24┊  {}, // initial state
+┊  ┊25┊  composeWithDevTools(),
+┊  ┊26┊);
+┊  ┊27┊
+┊  ┊28┊const cache = new ReduxCache({ store });
+┊  ┊29┊
+┊  ┊30┊const reduxLink = new ReduxLink(store);
+┊  ┊31┊
+┊  ┊32┊const httpLink = createHttpLink({ uri: `http://${URL}/graphql` });
+┊  ┊33┊
+┊  ┊34┊const link = ApolloLink.from([
+┊  ┊35┊  reduxLink,
+┊  ┊36┊  httpLink,
+┊  ┊37┊]);
+┊  ┊38┊
+┊  ┊39┊export const client = new ApolloClient({
+┊  ┊40┊  link,
+┊  ┊41┊  cache,
+┊  ┊42┊});
+┊  ┊43┊
 ┊ 8┊44┊const styles = StyleSheet.create({
 ┊ 9┊45┊  container: {
 ┊10┊46┊    flex: 1,
```

##### Changed client&#x2F;yarn.lock
```diff
@@ -2,6 +2,21 @@
 ┊ 2┊ 2┊# yarn lockfile v1
 ┊ 3┊ 3┊
 ┊ 4┊ 4┊
+┊  ┊ 5┊"@babel/runtime@^7.0.0-beta.32":
+┊  ┊ 6┊  version "7.0.0-beta.39"
+┊  ┊ 7┊  resolved "https://registry.yarnpkg.com/@babel/runtime/-/runtime-7.0.0-beta.39.tgz#e4b763b78d31a2c209165bae2772f103594a8972"
+┊  ┊ 8┊  dependencies:
+┊  ┊ 9┊    core-js "^2.5.3"
+┊  ┊10┊    regenerator-runtime "^0.11.1"
+┊  ┊11┊
+┊  ┊12┊"@types/async@2.0.47":
+┊  ┊13┊  version "2.0.47"
+┊  ┊14┊  resolved "https://registry.yarnpkg.com/@types/async/-/async-2.0.47.tgz#f49ba1dd1f189486beb6e1d070a850f6ab4bd521"
+┊  ┊15┊
+┊  ┊16┊"@types/zen-observable@0.5.3", "@types/zen-observable@^0.5.3":
+┊  ┊17┊  version "0.5.3"
+┊  ┊18┊  resolved "https://registry.yarnpkg.com/@types/zen-observable/-/zen-observable-0.5.3.tgz#91b728599544efbb7386d8b6633693a3c2e7ade5"
+┊  ┊19┊
 ┊ 5┊20┊abab@^1.0.3:
 ┊ 6┊21┊  version "1.0.4"
 ┊ 7┊22┊  resolved "https://registry.yarnpkg.com/abab/-/abab-1.0.4.tgz#5faad9c2c07f60dd76770f71cf025b62a63cfd4e"
```
```diff
@@ -103,6 +118,72 @@
 ┊103┊118┊    micromatch "^2.1.5"
 ┊104┊119┊    normalize-path "^2.0.0"
 ┊105┊120┊
+┊   ┊121┊apollo-cache-inmemory@^1.1.4:
+┊   ┊122┊  version "1.1.7"
+┊   ┊123┊  resolved "https://registry.yarnpkg.com/apollo-cache-inmemory/-/apollo-cache-inmemory-1.1.7.tgz#15e6200f70431414d29bd5f20e86d81739e26430"
+┊   ┊124┊  dependencies:
+┊   ┊125┊    apollo-cache "^1.1.2"
+┊   ┊126┊    apollo-utilities "^1.0.6"
+┊   ┊127┊    graphql-anywhere "^4.1.3"
+┊   ┊128┊
+┊   ┊129┊apollo-cache-redux@^0.1.0-alpha.7:
+┊   ┊130┊  version "0.1.0-alpha.7"
+┊   ┊131┊  resolved "https://registry.yarnpkg.com/apollo-cache-redux/-/apollo-cache-redux-0.1.0-alpha.7.tgz#30d4fddf6090447cb302177d39a01e5d7aeafdba"
+┊   ┊132┊  dependencies:
+┊   ┊133┊    apollo-cache "^1.0.2"
+┊   ┊134┊    apollo-cache-inmemory "^1.1.4"
+┊   ┊135┊
+┊   ┊136┊apollo-cache@^1.0.2, apollo-cache@^1.1.2:
+┊   ┊137┊  version "1.1.2"
+┊   ┊138┊  resolved "https://registry.yarnpkg.com/apollo-cache/-/apollo-cache-1.1.2.tgz#b1843a0e01d3837239e9925cfaa1d786599b77a9"
+┊   ┊139┊  dependencies:
+┊   ┊140┊    apollo-utilities "^1.0.6"
+┊   ┊141┊
+┊   ┊142┊apollo-client@^2.2.3:
+┊   ┊143┊  version "2.2.3"
+┊   ┊144┊  resolved "https://registry.yarnpkg.com/apollo-client/-/apollo-client-2.2.3.tgz#a8df51c9ff89acb0d98de81b911e56b1ce468ca3"
+┊   ┊145┊  dependencies:
+┊   ┊146┊    "@types/zen-observable" "^0.5.3"
+┊   ┊147┊    apollo-cache "^1.1.2"
+┊   ┊148┊    apollo-link "^1.0.0"
+┊   ┊149┊    apollo-link-dedup "^1.0.0"
+┊   ┊150┊    apollo-utilities "^1.0.6"
+┊   ┊151┊    symbol-observable "^1.0.2"
+┊   ┊152┊    zen-observable "^0.7.0"
+┊   ┊153┊  optionalDependencies:
+┊   ┊154┊    "@types/async" "2.0.47"
+┊   ┊155┊
+┊   ┊156┊apollo-link-dedup@^1.0.0:
+┊   ┊157┊  version "1.0.6"
+┊   ┊158┊  resolved "https://registry.yarnpkg.com/apollo-link-dedup/-/apollo-link-dedup-1.0.6.tgz#566ab421a5f6ef41995e2e386f575600d51b1408"
+┊   ┊159┊  dependencies:
+┊   ┊160┊    apollo-link "^1.1.0"
+┊   ┊161┊
+┊   ┊162┊apollo-link-http@^1.3.3:
+┊   ┊163┊  version "1.3.3"
+┊   ┊164┊  resolved "https://registry.yarnpkg.com/apollo-link-http/-/apollo-link-http-1.3.3.tgz#cb792c73266607e6361c8c1cc4dd42d405ca08f1"
+┊   ┊165┊  dependencies:
+┊   ┊166┊    apollo-link "^1.1.0"
+┊   ┊167┊
+┊   ┊168┊apollo-link-redux@^0.2.1:
+┊   ┊169┊  version "0.2.1"
+┊   ┊170┊  resolved "https://registry.yarnpkg.com/apollo-link-redux/-/apollo-link-redux-0.2.1.tgz#6b7a9b3f93264a1c8b03fe672d491479e8d92607"
+┊   ┊171┊  dependencies:
+┊   ┊172┊    "@babel/runtime" "^7.0.0-beta.32"
+┊   ┊173┊    apollo-utilities "^1.0.0"
+┊   ┊174┊
+┊   ┊175┊apollo-link@^1.0.0, apollo-link@^1.1.0:
+┊   ┊176┊  version "1.1.0"
+┊   ┊177┊  resolved "https://registry.yarnpkg.com/apollo-link/-/apollo-link-1.1.0.tgz#9d573b16387ee0d8e147b1f319e42c8c562f18f7"
+┊   ┊178┊  dependencies:
+┊   ┊179┊    "@types/zen-observable" "0.5.3"
+┊   ┊180┊    apollo-utilities "^1.0.0"
+┊   ┊181┊    zen-observable "^0.7.0"
+┊   ┊182┊
+┊   ┊183┊apollo-utilities@^1.0.0, apollo-utilities@^1.0.6:
+┊   ┊184┊  version "1.0.6"
+┊   ┊185┊  resolved "https://registry.yarnpkg.com/apollo-utilities/-/apollo-utilities-1.0.6.tgz#7bfd7a702b5225c9a4591fe28c5899d9b5f08889"
+┊   ┊186┊
 ┊106┊187┊append-transform@^0.4.0:
 ┊107┊188┊  version "0.4.0"
 ┊108┊189┊  resolved "https://registry.yarnpkg.com/append-transform/-/append-transform-0.4.0.tgz#d76ebf8ca94d276e247a36bad44a4b74ab611991"
```
```diff
@@ -1289,7 +1370,7 @@
 ┊1289┊1370┊  version "2.4.1"
 ┊1290┊1371┊  resolved "https://registry.yarnpkg.com/core-js/-/core-js-2.4.1.tgz#4de911e667b0eae9124e34254b53aea6fc618d3e"
 ┊1291┊1372┊
-┊1292┊    ┊core-js@^2.4.0, core-js@^2.4.1, core-js@^2.5.0:
+┊    ┊1373┊core-js@^2.4.0, core-js@^2.4.1, core-js@^2.5.0, core-js@^2.5.3:
 ┊1293┊1374┊  version "2.5.3"
 ┊1294┊1375┊  resolved "https://registry.yarnpkg.com/core-js/-/core-js-2.5.3.tgz#8acc38345824f16d8365b7c9b4259168e8ed603e"
 ┊1295┊1376┊
```
```diff
@@ -1662,7 +1743,7 @@
 ┊1662┊1743┊    semver "^5.1.0"
 ┊1663┊1744┊    through2 "^2.0.0"
 ┊1664┊1745┊
-┊1665┊    ┊fbjs@^0.8.14, fbjs@^0.8.16:
+┊    ┊1746┊fbjs@^0.8.14, fbjs@^0.8.16, fbjs@^0.8.9:
 ┊1666┊1747┊  version "0.8.16"
 ┊1667┊1748┊  resolved "https://registry.yarnpkg.com/fbjs/-/fbjs-0.8.16.tgz#5e67432f550dc41b572bf55847b8aca64e5337db"
 ┊1668┊1749┊  dependencies:
```
```diff
@@ -1674,18 +1755,6 @@
 ┊1674┊1755┊    setimmediate "^1.0.5"
 ┊1675┊1756┊    ua-parser-js "^0.7.9"
 ┊1676┊1757┊
-┊1677┊    ┊fbjs@^0.8.9:
-┊1678┊    ┊  version "0.8.14"
-┊1679┊    ┊  resolved "https://registry.yarnpkg.com/fbjs/-/fbjs-0.8.14.tgz#d1dbe2be254c35a91e09f31f9cd50a40b2a0ed1c"
-┊1680┊    ┊  dependencies:
-┊1681┊    ┊    core-js "^1.0.0"
-┊1682┊    ┊    isomorphic-fetch "^2.1.1"
-┊1683┊    ┊    loose-envify "^1.0.0"
-┊1684┊    ┊    object-assign "^4.1.0"
-┊1685┊    ┊    promise "^7.1.1"
-┊1686┊    ┊    setimmediate "^1.0.5"
-┊1687┊    ┊    ua-parser-js "^0.7.9"
-┊1688┊    ┊
 ┊1689┊1758┊figures@^2.0.0:
 ┊1690┊1759┊  version "2.0.0"
 ┊1691┊1760┊  resolved "https://registry.yarnpkg.com/figures/-/figures-2.0.0.tgz#3ab1a2d2a62c8bfb431a0c94cb797a2fce27c962"
```
```diff
@@ -1887,6 +1956,22 @@
 ┊1887┊1956┊  version "4.1.11"
 ┊1888┊1957┊  resolved "https://registry.yarnpkg.com/graceful-fs/-/graceful-fs-4.1.11.tgz#0e8bdfe4d1ddb8854d64e04ea7c00e2a026e5658"
 ┊1889┊1958┊
+┊    ┊1959┊graphql-anywhere@^4.1.3:
+┊    ┊1960┊  version "4.1.3"
+┊    ┊1961┊  resolved "https://registry.yarnpkg.com/graphql-anywhere/-/graphql-anywhere-4.1.3.tgz#ddd857d45d1538f55e8364c6c7a9016817a5ea92"
+┊    ┊1962┊  dependencies:
+┊    ┊1963┊    apollo-utilities "^1.0.6"
+┊    ┊1964┊
+┊    ┊1965┊graphql-tag@^2.4.2:
+┊    ┊1966┊  version "2.7.3"
+┊    ┊1967┊  resolved "https://registry.yarnpkg.com/graphql-tag/-/graphql-tag-2.7.3.tgz#5040112a1b4623285ef017c252276f0dea37f03f"
+┊    ┊1968┊
+┊    ┊1969┊graphql@^0.12.3:
+┊    ┊1970┊  version "0.12.3"
+┊    ┊1971┊  resolved "https://registry.yarnpkg.com/graphql/-/graphql-0.12.3.tgz#11668458bbe28261c0dcb6e265f515ba79f6ce07"
+┊    ┊1972┊  dependencies:
+┊    ┊1973┊    iterall "1.1.3"
+┊    ┊1974┊
 ┊1890┊1975┊growly@^1.3.0:
 ┊1891┊1976┊  version "1.3.0"
 ┊1892┊1977┊  resolved "https://registry.yarnpkg.com/growly/-/growly-1.3.0.tgz#f10748cbe76af964b7c96c93c6bcc28af120c081"
```
```diff
@@ -2002,6 +2087,10 @@
 ┊2002┊2087┊  version "4.2.0"
 ┊2003┊2088┊  resolved "https://registry.yarnpkg.com/hoek/-/hoek-4.2.0.tgz#72d9d0754f7fe25ca2d01ad8f8f9a9449a89526d"
 ┊2004┊2089┊
+┊    ┊2090┊hoist-non-react-statics@^2.2.0, hoist-non-react-statics@^2.2.1:
+┊    ┊2091┊  version "2.3.1"
+┊    ┊2092┊  resolved "https://registry.yarnpkg.com/hoist-non-react-statics/-/hoist-non-react-statics-2.3.1.tgz#343db84c6018c650778898240135a1420ee22ce0"
+┊    ┊2093┊
 ┊2005┊2094┊home-or-tmp@^2.0.0:
 ┊2006┊2095┊  version "2.0.0"
 ┊2007┊2096┊  resolved "https://registry.yarnpkg.com/home-or-tmp/-/home-or-tmp-2.0.0.tgz#e36c3f2d2cae7d746a857e38d18d5f32a7882db8"
```
```diff
@@ -2050,11 +2139,11 @@
 ┊2050┊2139┊  version "0.4.13"
 ┊2051┊2140┊  resolved "https://registry.yarnpkg.com/iconv-lite/-/iconv-lite-0.4.13.tgz#1f88aba4ab0b1508e8312acc39345f36e992e2f2"
 ┊2052┊2141┊
-┊2053┊    ┊iconv-lite@0.4.19, iconv-lite@^0.4.8:
+┊    ┊2142┊iconv-lite@0.4.19, iconv-lite@^0.4.8, iconv-lite@~0.4.13:
 ┊2054┊2143┊  version "0.4.19"
 ┊2055┊2144┊  resolved "https://registry.yarnpkg.com/iconv-lite/-/iconv-lite-0.4.19.tgz#f7468f60135f5e5dad3399c0a81be9a1603a082b"
 ┊2056┊2145┊
-┊2057┊    ┊iconv-lite@^0.4.17, iconv-lite@~0.4.13:
+┊    ┊2146┊iconv-lite@^0.4.17:
 ┊2058┊2147┊  version "0.4.18"
 ┊2059┊2148┊  resolved "https://registry.yarnpkg.com/iconv-lite/-/iconv-lite-0.4.18.tgz#23d8656b16aae6742ac29732ea8f0336a4789cf2"
 ┊2060┊2149┊
```
```diff
@@ -2100,7 +2189,7 @@
 ┊2100┊2189┊    strip-ansi "^4.0.0"
 ┊2101┊2190┊    through "^2.3.6"
 ┊2102┊2191┊
-┊2103┊    ┊invariant@^2.2.0, invariant@^2.2.2:
+┊    ┊2192┊invariant@^2.0.0, invariant@^2.2.0, invariant@^2.2.1, invariant@^2.2.2:
 ┊2104┊2193┊  version "2.2.2"
 ┊2105┊2194┊  resolved "https://registry.yarnpkg.com/invariant/-/invariant-2.2.2.tgz#9e1f56ac0acdb6bf303306f338be3b204ae60360"
 ┊2106┊2195┊  dependencies:
```
```diff
@@ -2298,6 +2387,10 @@
 ┊2298┊2387┊  dependencies:
 ┊2299┊2388┊    handlebars "^4.0.3"
 ┊2300┊2389┊
+┊    ┊2390┊iterall@1.1.3:
+┊    ┊2391┊  version "1.1.3"
+┊    ┊2392┊  resolved "https://registry.yarnpkg.com/iterall/-/iterall-1.1.3.tgz#1cbbff96204056dde6656e2ed2e2226d0e6d72c9"
+┊    ┊2393┊
 ┊2301┊2394┊jest-changed-files@^20.0.3:
 ┊2302┊2395┊  version "20.0.3"
 ┊2303┊2396┊  resolved "https://registry.yarnpkg.com/jest-changed-files/-/jest-changed-files-20.0.3.tgz#9394d5cc65c438406149bef1bf4d52b68e03e3f8"
```
```diff
@@ -2709,6 +2802,10 @@
 ┊2709┊2802┊    p-locate "^2.0.0"
 ┊2710┊2803┊    path-exists "^3.0.0"
 ┊2711┊2804┊
+┊    ┊2805┊lodash-es@^4.2.0, lodash-es@^4.2.1:
+┊    ┊2806┊  version "4.17.4"
+┊    ┊2807┊  resolved "https://registry.yarnpkg.com/lodash-es/-/lodash-es-4.17.4.tgz#dcc1d7552e150a0640073ba9cb31d70f032950e7"
+┊    ┊2808┊
 ┊2712┊2809┊lodash._basecopy@^3.0.0:
 ┊2713┊2810┊  version "3.0.1"
 ┊2714┊2811┊  resolved "https://registry.yarnpkg.com/lodash._basecopy/-/lodash._basecopy-3.0.1.tgz#8da0e6a876cf344c0ad8a54882111dd3c5c7ca36"
```
```diff
@@ -2751,6 +2848,10 @@
 ┊2751┊2848┊  dependencies:
 ┊2752┊2849┊    lodash._root "^3.0.0"
 ┊2753┊2850┊
+┊    ┊2851┊lodash.flowright@^3.5.0:
+┊    ┊2852┊  version "3.5.0"
+┊    ┊2853┊  resolved "https://registry.yarnpkg.com/lodash.flowright/-/lodash.flowright-3.5.0.tgz#2b5fff399716d7e7dc5724fe9349f67065184d67"
+┊    ┊2854┊
 ┊2754┊2855┊lodash.isarguments@^3.0.0:
 ┊2755┊2856┊  version "3.1.0"
 ┊2756┊2857┊  resolved "https://registry.yarnpkg.com/lodash.isarguments/-/lodash.isarguments-3.1.0.tgz#2f573d85c6a24289ff00663b491c1d338ff3458a"
```
```diff
@@ -2779,6 +2880,10 @@
 ┊2779┊2880┊  version "4.6.1"
 ┊2780┊2881┊  resolved "https://registry.yarnpkg.com/lodash.padstart/-/lodash.padstart-4.6.1.tgz#d2e3eebff0d9d39ad50f5cbd1b52a7bce6bb611b"
 ┊2781┊2882┊
+┊    ┊2883┊lodash.pick@^4.4.0:
+┊    ┊2884┊  version "4.4.0"
+┊    ┊2885┊  resolved "https://registry.yarnpkg.com/lodash.pick/-/lodash.pick-4.4.0.tgz#52f05610fff9ded422611441ed1fc123a03001b3"
+┊    ┊2886┊
 ┊2782┊2887┊lodash.restparam@^3.0.0:
 ┊2783┊2888┊  version "3.6.1"
 ┊2784┊2889┊  resolved "https://registry.yarnpkg.com/lodash.restparam/-/lodash.restparam-3.6.1.tgz#936a4e309ef330a7645ed4145986c85ae5b20805"
```
```diff
@@ -2808,7 +2913,7 @@
 ┊2808┊2913┊  version "3.10.1"
 ┊2809┊2914┊  resolved "https://registry.yarnpkg.com/lodash/-/lodash-3.10.1.tgz#5bf45e8e49ba4189e17d482789dfd15bd140b7b6"
 ┊2810┊2915┊
-┊2811┊    ┊lodash@^4.14.0, lodash@^4.16.6, lodash@^4.17.4, lodash@^4.2.0, lodash@^4.3.0, lodash@^4.6.1:
+┊    ┊2916┊lodash@^4.14.0, lodash@^4.16.6, lodash@^4.17.4, lodash@^4.2.0, lodash@^4.2.1, lodash@^4.3.0, lodash@^4.6.1:
 ┊2812┊2917┊  version "4.17.4"
 ┊2813┊2918┊  resolved "https://registry.yarnpkg.com/lodash/-/lodash-4.17.4.tgz#78203a4d1c328ae1d86dca6460e369b57f4055ae"
 ┊2814┊2919┊
```
```diff
@@ -3074,7 +3179,14 @@
 ┊3074┊3179┊  version "0.6.1"
 ┊3075┊3180┊  resolved "https://registry.yarnpkg.com/negotiator/-/negotiator-0.6.1.tgz#2b327184e8992101177b28563fb5e7102acd0ca9"
 ┊3076┊3181┊
-┊3077┊    ┊node-fetch@^1.0.1, node-fetch@^1.3.3:
+┊    ┊3182┊node-fetch@^1.0.1:
+┊    ┊3183┊  version "1.7.3"
+┊    ┊3184┊  resolved "https://registry.yarnpkg.com/node-fetch/-/node-fetch-1.7.3.tgz#980f6f72d85211a5347c6b2bc18c5b84c3eb47ef"
+┊    ┊3185┊  dependencies:
+┊    ┊3186┊    encoding "^0.1.11"
+┊    ┊3187┊    is-stream "^1.0.1"
+┊    ┊3188┊
+┊    ┊3189┊node-fetch@^1.3.3:
 ┊3078┊3190┊  version "1.7.1"
 ┊3079┊3191┊  resolved "https://registry.yarnpkg.com/node-fetch/-/node-fetch-1.7.1.tgz#899cb3d0a3c92f952c47f1b876f4c8aeabd400d5"
 ┊3080┊3192┊  dependencies:
```
```diff
@@ -3433,14 +3545,7 @@
 ┊3433┊3545┊  dependencies:
 ┊3434┊3546┊    asap "~2.0.3"
 ┊3435┊3547┊
-┊3436┊    ┊prop-types@^15.5.8:
-┊3437┊    ┊  version "15.5.10"
-┊3438┊    ┊  resolved "https://registry.yarnpkg.com/prop-types/-/prop-types-15.5.10.tgz#2797dfc3126182e3a95e3dfbb2e893ddd7456154"
-┊3439┊    ┊  dependencies:
-┊3440┊    ┊    fbjs "^0.8.9"
-┊3441┊    ┊    loose-envify "^1.3.1"
-┊3442┊    ┊
-┊3443┊    ┊prop-types@^15.6.0:
+┊    ┊3548┊prop-types@^15.5.10, prop-types@^15.6.0:
 ┊3444┊3549┊  version "15.6.0"
 ┊3445┊3550┊  resolved "https://registry.yarnpkg.com/prop-types/-/prop-types-15.6.0.tgz#ceaf083022fc46b4a35f69e13ef75aed0d639856"
 ┊3446┊3551┊  dependencies:
```
```diff
@@ -3448,6 +3553,13 @@
 ┊3448┊3553┊    loose-envify "^1.3.1"
 ┊3449┊3554┊    object-assign "^4.1.1"
 ┊3450┊3555┊
+┊    ┊3556┊prop-types@^15.5.8:
+┊    ┊3557┊  version "15.5.10"
+┊    ┊3558┊  resolved "https://registry.yarnpkg.com/prop-types/-/prop-types-15.5.10.tgz#2797dfc3126182e3a95e3dfbb2e893ddd7456154"
+┊    ┊3559┊  dependencies:
+┊    ┊3560┊    fbjs "^0.8.9"
+┊    ┊3561┊    loose-envify "^1.3.1"
+┊    ┊3562┊
 ┊3451┊3563┊prr@~1.0.1:
 ┊3452┊3564┊  version "1.0.1"
 ┊3453┊3565┊  resolved "https://registry.yarnpkg.com/prr/-/prr-1.0.1.tgz#d3fc114ba06995a45ec6893f484ceb1d78f5f476"
```
```diff
@@ -3504,6 +3616,17 @@
 ┊3504┊3616┊    minimist "^1.2.0"
 ┊3505┊3617┊    strip-json-comments "~2.0.1"
 ┊3506┊3618┊
+┊    ┊3619┊react-apollo@^2.0.4:
+┊    ┊3620┊  version "2.0.4"
+┊    ┊3621┊  resolved "https://registry.yarnpkg.com/react-apollo/-/react-apollo-2.0.4.tgz#01dd32a8e388672f5d7385b21cdd0b94009ee9ee"
+┊    ┊3622┊  dependencies:
+┊    ┊3623┊    apollo-link "^1.0.0"
+┊    ┊3624┊    hoist-non-react-statics "^2.2.0"
+┊    ┊3625┊    invariant "^2.2.1"
+┊    ┊3626┊    lodash.flowright "^3.5.0"
+┊    ┊3627┊    lodash.pick "^4.4.0"
+┊    ┊3628┊    prop-types "^15.5.8"
+┊    ┊3629┊
 ┊3507┊3630┊react-clone-referenced-element@^1.0.1:
 ┊3508┊3631┊  version "1.0.1"
 ┊3509┊3632┊  resolved "https://registry.yarnpkg.com/react-clone-referenced-element/-/react-clone-referenced-element-1.0.1.tgz#2bba8c69404c5e4a944398600bcc4c941f860682"
```
```diff
@@ -3585,6 +3708,17 @@
 ┊3585┊3708┊    lodash "^4.6.1"
 ┊3586┊3709┊    react-deep-force-update "^1.0.0"
 ┊3587┊3710┊
+┊    ┊3711┊react-redux@^5.0.5:
+┊    ┊3712┊  version "5.0.6"
+┊    ┊3713┊  resolved "https://registry.yarnpkg.com/react-redux/-/react-redux-5.0.6.tgz#23ed3a4f986359d68b5212eaaa681e60d6574946"
+┊    ┊3714┊  dependencies:
+┊    ┊3715┊    hoist-non-react-statics "^2.2.1"
+┊    ┊3716┊    invariant "^2.0.0"
+┊    ┊3717┊    lodash "^4.2.0"
+┊    ┊3718┊    lodash-es "^4.2.0"
+┊    ┊3719┊    loose-envify "^1.1.0"
+┊    ┊3720┊    prop-types "^15.5.10"
+┊    ┊3721┊
 ┊3588┊3722┊react-test-renderer@16.0.0-alpha.12:
 ┊3589┊3723┊  version "16.0.0-alpha.12"
 ┊3590┊3724┊  resolved "https://registry.yarnpkg.com/react-test-renderer/-/react-test-renderer-16.0.0-alpha.12.tgz#9e4cc5d8ce8bfca72778340de3e1454b9d6c0cc5"
```
```diff
@@ -3663,6 +3797,19 @@
 ┊3663┊3797┊    isarray "0.0.1"
 ┊3664┊3798┊    string_decoder "~0.10.x"
 ┊3665┊3799┊
+┊    ┊3800┊redux-devtools-extension@^2.13.2:
+┊    ┊3801┊  version "2.13.2"
+┊    ┊3802┊  resolved "https://registry.yarnpkg.com/redux-devtools-extension/-/redux-devtools-extension-2.13.2.tgz#e0f9a8e8dfca7c17be92c7124958a3b94eb2911d"
+┊    ┊3803┊
+┊    ┊3804┊redux@^3.7.2:
+┊    ┊3805┊  version "3.7.2"
+┊    ┊3806┊  resolved "https://registry.yarnpkg.com/redux/-/redux-3.7.2.tgz#06b73123215901d25d065be342eb026bc1c8537b"
+┊    ┊3807┊  dependencies:
+┊    ┊3808┊    lodash "^4.2.1"
+┊    ┊3809┊    lodash-es "^4.2.1"
+┊    ┊3810┊    loose-envify "^1.1.0"
+┊    ┊3811┊    symbol-observable "^1.0.3"
+┊    ┊3812┊
 ┊3666┊3813┊regenerate@^1.2.1:
 ┊3667┊3814┊  version "1.3.2"
 ┊3668┊3815┊  resolved "https://registry.yarnpkg.com/regenerate/-/regenerate-1.3.2.tgz#d1941c67bad437e1be76433add5b385f95b19260"
```
```diff
@@ -3671,7 +3818,7 @@
 ┊3671┊3818┊  version "0.10.5"
 ┊3672┊3819┊  resolved "https://registry.yarnpkg.com/regenerator-runtime/-/regenerator-runtime-0.10.5.tgz#336c3efc1220adcedda2c9fab67b5a7955a33658"
 ┊3673┊3820┊
-┊3674┊    ┊regenerator-runtime@^0.11.0:
+┊    ┊3821┊regenerator-runtime@^0.11.0, regenerator-runtime@^0.11.1:
 ┊3675┊3822┊  version "0.11.1"
 ┊3676┊3823┊  resolved "https://registry.yarnpkg.com/regenerator-runtime/-/regenerator-runtime-0.11.1.tgz#be05ad7f9bf7d22e056f9726cee5017fbf19e2e9"
 ┊3677┊3824┊
```
```diff
@@ -4171,6 +4318,14 @@
 ┊4171┊4318┊  dependencies:
 ┊4172┊4319┊    has-flag "^2.0.0"
 ┊4173┊4320┊
+┊    ┊4321┊symbol-observable@^1.0.2:
+┊    ┊4322┊  version "1.2.0"
+┊    ┊4323┊  resolved "https://registry.yarnpkg.com/symbol-observable/-/symbol-observable-1.2.0.tgz#c22688aed4eab3cdc2dfeacbb561660560a00804"
+┊    ┊4324┊
+┊    ┊4325┊symbol-observable@^1.0.3:
+┊    ┊4326┊  version "1.0.4"
+┊    ┊4327┊  resolved "https://registry.yarnpkg.com/symbol-observable/-/symbol-observable-1.0.4.tgz#29bf615d4aa7121bdd898b22d4b3f9bc4e2aa03d"
+┊    ┊4328┊
 ┊4174┊4329┊symbol-tree@^3.2.1:
 ┊4175┊4330┊  version "3.2.2"
 ┊4176┊4331┊  resolved "https://registry.yarnpkg.com/symbol-tree/-/symbol-tree-3.2.2.tgz#ae27db38f660a7ae2e1c3b7d1bc290819b8519e6"
```
```diff
@@ -4300,8 +4455,8 @@
 ┊4300┊4455┊  resolved "https://registry.yarnpkg.com/typedarray/-/typedarray-0.0.6.tgz#867ac74e3864187b1d3d47d996a78ec5c8830777"
 ┊4301┊4456┊
 ┊4302┊4457┊ua-parser-js@^0.7.9:
-┊4303┊    ┊  version "0.7.14"
-┊4304┊    ┊  resolved "https://registry.yarnpkg.com/ua-parser-js/-/ua-parser-js-0.7.14.tgz#110d53fa4c3f326c121292bbeac904d2e03387ca"
+┊    ┊4458┊  version "0.7.17"
+┊    ┊4459┊  resolved "https://registry.yarnpkg.com/ua-parser-js/-/ua-parser-js-0.7.17.tgz#e9ec5f9498b9ec910e7ae3ac626a805c4d09ecac"
 ┊4305┊4460┊
 ┊4306┊4461┊uglify-es@^3.1.9:
 ┊4307┊4462┊  version "3.3.6"
```
```diff
@@ -4627,3 +4782,7 @@
 ┊4627┊4782┊    cliui "^2.1.0"
 ┊4628┊4783┊    decamelize "^1.0.0"
 ┊4629┊4784┊    window-size "0.1.0"
+┊    ┊4785┊
+┊    ┊4786┊zen-observable@^0.7.0:
+┊    ┊4787┊  version "0.7.1"
+┊    ┊4788┊  resolved "https://registry.yarnpkg.com/zen-observable/-/zen-observable-0.7.1.tgz#f84075c0ee085594d3566e1d6454207f126411b3"
```

[}]: #

Finally, we wrap our `App` component in the `ApolloProvider` component from `react-apollo`. `ApolloProvider` connects our app to Redux and Apollo at the same time.

[{]: <helper> (diffStep 1.11)

#### Step 1.11: Add ApolloProvider to App

##### Changed client&#x2F;src&#x2F;app.js
```diff
@@ -63,18 +63,22 @@
 ┊63┊63┊export default class App extends Component {
 ┊64┊64┊  render() {
 ┊65┊65┊    return (
-┊66┊  ┊      <View style={styles.container}>
-┊67┊  ┊        <Text style={styles.welcome}>
-┊68┊  ┊          Welcome to React Native!
-┊69┊  ┊        </Text>
-┊70┊  ┊        <Text style={styles.instructions}>
-┊71┊  ┊          To get started, edit index.ios.js
-┊72┊  ┊        </Text>
-┊73┊  ┊        <Text style={styles.instructions}>
-┊74┊  ┊          Press Cmd+R to reload,{'\n'}
-┊75┊  ┊          Cmd+D or shake for dev menu
-┊76┊  ┊        </Text>
-┊77┊  ┊      </View>
+┊  ┊66┊      <ApolloProvider client={client}>
+┊  ┊67┊        <Provider store={store}>
+┊  ┊68┊          <View style={styles.container}>
+┊  ┊69┊            <Text style={styles.welcome}>
+┊  ┊70┊              Welcome to React Native!
+┊  ┊71┊            </Text>
+┊  ┊72┊            <Text style={styles.instructions}>
+┊  ┊73┊              To get started, edit index.ios.js
+┊  ┊74┊            </Text>
+┊  ┊75┊            <Text style={styles.instructions}>
+┊  ┊76┊              Press Cmd+R to reload,{'\n'}
+┊  ┊77┊              Cmd+D or shake for dev menu
+┊  ┊78┊            </Text>
+┊  ┊79┊          </View>
+┊  ┊80┊        </Provider>
+┊  ┊81┊      </ApolloProvider>
 ┊78┊82┊    );
 ┊79┊83┊  }
 ┊80┊84┊}
```

[}]: #

If we reload the app `(CMD + R)`, there hopefully should be no errors in the simulator. We can check if everything is hooked up properly by opening Redux Native Debugger and confirming the Redux store includes `apollo`: ![Redux Devtools Image](https://s3-us-west-1.amazonaws.com/tortilla/chatty/step1-11.png)

[{]: <helper> (navStep)

| [< Intro](../../../README.md) | [Next Step >](step2.md) |
|:--------------------------------|--------------------------------:|

[}]: #
