---
layout: post
title: "Fixing React Native's AccessibilityInfo Module Error"
date: 2017-08-07
---

Have you ever been hit with this awful, horrifying red screen error while developing for mobile in React Native?
![Accessibility Info Red Screen Error]({{ site.url }}/assets/accessibility-info-error.png)

## Do Not Despair! 
I spent many hours over the past couple of weeks dealing with this specific error and understanding what causes it, and am here to help!
There is a very long github issue which can be found [here](https://github.com/facebook/react-native/issues/14209). I am going to go into one particular scenario that causes this error to arise, but in many cases, deleting your `node_modules` directory and then reinstalling with `npm install` or `yarn install` and then resetting the cache with `npm start -- --reset-cache`.

**BEWARE**. The `--` between `npm start` and `--reset-cache` is **CRUCIAL**.  
`npm start` is in fact an alias for `npm run start`, so if you omit this `--` then the `--reset-cache` option is actually being applied to `npm run`, where it has no effect and does not actually reset the cache.

## The Extra Tricky Scenario
Our problem arose because we were essentially creating an npm package that we wanted to test with a basic skeleton app, which was positioned in the directory. If we call our directory with our source code for our package `dirA` and our example app `RNExample`, this is the directory structure:
```
- dirA
  - src
  - dist
  - RNExample
    - index.js
    - node_modules
     - dirA
```
Then the following would happen once I ran `yarn install` in both the parent `dirA` directory and the inner `RNExample` directory:

```
- dirA
  - src
  - dist
  - node_modules
  	- react-native
  - RNExample
    - index.js
    - node_modules
      - react-native
      - dirA
        - node_modules
          - react-native
```
Essentially, what was happening was that the linking was not properly happening, and we were ending up with two copies of the `react-native` package under `RNExample`, confusing the builder. In fact, when you run `npm start -- --reset-cache`, the console should give you a warning suggesting that modules exist in two different locations.

There are two possible fixes: one short term hack, and one better long term solution.

## The Short Term Hack
Once we run `yarn install` in both `dirA` and `dirA/RNExample`, and end up with the above directory structure, simply run `rm -rf node_modules/dirA/node_modules/react-native` from inside `RNExample`. This should ensure that the app only runs with the outer `react-native` module.

## The Long Term Solution
Update the `package.json` file inside `dirA` so that `react-native` is a peerDependency. This means `package.json` should look something like this:
```json
{
  "name": "dirA",
  "author": "Reuben Abraham",
  "dependencies": {
    ...
  },
  "devDependencies": {
    ...
  },
  "peerDependencies": {
    "react": (version-here),
    "react-dom": (version-here),
    "react-native": (version-here)
  }
}
```
Then run `yarn cache clean` to stop how yarn previously cached the dependencies. Then from inside `dirA`, run `yarn install; cd RNExample; yarn install`. Double check that you don't have the second `react-native` module inside `RNExample/node_modules/dirA/node_modules`. Finally run `npm start -- --reset-cache` and you should be good to go!

Hopefully this saves all of you a few dev hours that you can use more productively!
