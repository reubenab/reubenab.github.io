---
layout: post
title: "Improving React Redux App Performance"
date: 2018-07-30
---

A React Native mobile app I am working on faced significant performance issues (particularly on initial app load) that appeared to be largely attributable to excessive app re-renders. While investigating, I noticed that each of the five tabs that are initially mounted were re-rendering on every single state change of the Redux store. This problem is especially acute on initial app load because we have a number of disparate requests to fetch various pieces of data for the different tab screens that each modify the state object on success. I uncovered the major source of these renders to be an anti-pattern with the way we were writing and using selectors for various components to have access to slices of the state object. While this is still a source of tech debt, we were able to reduce time-to-interaction for users by approximately a second on an iPhone X by fixing a number of the selectors. 

*Note: though the work I did was on a React Native app, the learnings apply to any React Redux web app, too.*

## How React-Redux’s Connect Function Works
The connect function returns a wrapped (by default pure) component that has a number of performance optimizations out of the box, the most important one being a shallow compare of the combined props object after the state change against the one before the state change. By default as seen in the [source code](https://github.com/reactjs/react-redux/blob/f892ec00d7e92ff7afb21498276472f0e3b000c5/src/connect/selectorFactory.js#L42-L50), if `mapDispatchToProps` does not depend on the Component’s own props, the cached version is used on every state change, since dispatch does not change, but `mapStateToProps` is not cached, since the state object can change.

This means that when the state object updates, since a connected component is subscribed to this update, it runs its `mapStateToProps` and `mapDispatchToProps` functions, combines the results with the props provided from the parent components, and then shallow compares the result with the old set of props when deciding whether or not to trigger a re-render. 

Next, I will explore some common pitfalls that cause unnecessary re-renders that slow down your application.

*Note: some of this behavior can be overridden with the inclusion of certain options as an argument to the `connect` function.*

## Returning Default Objects and Arrays in Selectors
I noticed a number of selectors that did something like this:

```
const getX = state => _.get(state, ‘x’, {});
```

This has the advantage of letting components not worry about checking for `undefined` values, but unfortunately has poor performance implications. Let’s say that `mapStateToProps` then did this:

```
const mapStateToProps = state => ({ x: getX(state) });
```

Now, when the shallow compare happens in the `shouldComponentUpdate` lifecycle function of the connectHOC (the wrapper component returned by connect), we find that `prevProps.x !== this.props.x` since we have returned a new object in the selector for `x` every single time the Redux state changes and `mapStateToProps` is invoked. This object occupies a new space in memory, and so the component will re-render on every app-level state change when it is mounted. 

I have found a couple of ways around this:
- Use a frozen empty object that can be imported anywhere in your app. E.g. `const EMPTY_OBJECT = Object.freeze({});`.
- Do not use a default value and ensure that your components know how to handle `undefined` or `null` values. 

In my opinion, there is a distinction between `undefined` and an empty object/array, so number 2 is my preferred approach, but it may take time to refactor your components to handle this change, so the shorter term fix may be number 1.

## Passing New Objects/Arrays as Props
If a parent component passes props down to a child component such that it is a new object/array/function reference every single time render is called, then the child will also render even if none of its props have changed in structure. This eliminates the performance benefits gained from `PureComponents` and will also trigger re-renders on every state change of store connected components, since Redux’s `connect` will see new props having come down. The following is an example of this:

```javascript
render() {
  return (
    <YodleeFormInput 
      onTextChange={text => this.setState({ text })} 
    />
  );
}
```

To avoid this issue, simply write all such functions as arrow class functions so that they are auto-bound to the component once.

```javascript
setText = text => this.setState({ text });

render() {
  return (
    <YodleeFormInput onTextChange={this.setText} />
  );
}
```

## Computing Derived Data Using Transform Functions
Imagine the store looked something like this:

```javascript
{
  transactions: [
    { id: 1, amount: 100, status: 'POSTED' },
    { id: 2, amount: 50, status: 'PENDING'},
    { id: 3, amount: 80, status: 'POSTED' },
  ],
}
```

Then say we had one component that shows us all posted transactions. We could write a simple selector to use in `mapStateToProps` that does something like this:

```javascript
const getPostedTransactions = state => _.filter(
  state.transactions, 
  txn => txn.status === 'POSTED'
);
```

However, most of Lodash’s transform functions return new objects/arrays in memory. They are pure functions and do not modify the inputs (as they should not, we definitely do not want to change the state object). Therefore, once again, every time the state object changes and `mapStateToProps` is invoked, we get a new array for our `postedTransactions` prop. The official Redux documentation gives another similar example [here](https://redux.js.org/faq/react-redux#why-is-my-component-re-rendering-too-often) using a `map` function.

## Ways to Compute Derived Data More Efficiently

### Perform the Computations in Reducers
If we do these transformations in a reducer, they only happen once when we get new data from an action. Then, we could use a simple getter as a selector that just returns a slice of the state called `state.postedTransactions`. However, what if we had a component that wanted all transactions, and another component that wanted pending transactions. We could end up with a lot of duplicated data in the store, which is not terrible but perhaps you want to avoid that.

Another instance where this doesn’t work is if the data that you are trying to put in one slice of the store relies on some data from another slice of the store. For example, say one of the slices of your store held your feature flags, and your transactions slice somehow filtered based on a certain feature flag value. The reducer for the transactions slice does not have access to the feature flags slice.

### Perform the Computations in Components Using Utility Functions
We could write all our selectors to just return slices of the state untransformed. Then we could write a utility function that the component uses when deciding how to render to transform the object to its desired format. For example we could have a utility function that looks like this:

```javascript
const getPostedTransactionsFromAllTransactions = 
  transactions => _.filter(
    transactions, 
    txn => txn.status === 'POSTED'
  );
```

Then our selector we use in `mapStateToProps` would just be the following:

```javascript
const getTransactions = state => state.transactions;
```

The `render` function for our component would then have to do something like this:

```javascript
render() {
  const { transactions } = this.props;
  const postedTransactions = 
    getPostedTransactionsFromAllTransactions(transactions);
  return (
    <div>
      {_.map(postedTransactions, txn => 
        <Transaction txn={txn} />
      )}
    </div>
  );
}
```

The advantage here is that this prop is untransformed and so remains the same on every state change, until it actually changes. Another advantage of this is that you could also pass the relevant feature flag down to the component and use it to filter the transactions if you needed, which was a shortcoming with the reducer solution. 

One disadvantage of this approach is the following. Say that the component also had some prop called `x`. If `x` changed but transactions did not, the `render` function would still be called, which in turn would call the `getPostedTransactionsFromAllTransactions` function, even though the input to that function has not changed. If the utility function you use is expensive and/or other props are changing frequently, then this may have negative performance implications. The best way around this is to memoize the utility function. There are a number of packages that offer easy ways to memoize - reselect (discussed below) offers a default option that can be used as follows:

```javascript
import { defaultMemoize as memoize } from 'reselect';

...

const getPostedTransactionsFromAllTransactions = 
  memoize(transactions => 
    _.filter(transactions, txn => txn.status === 'POSTED')
  );
```

Then in your `render` function, you would render based on `this.props.transactions`.

This ensures that we always return the same object in memory from this function unless its underlying argument changes. This helps all child components from re-rendering as long as they are `PureComponents`.

One potential issue is that this could lead to more bloated components. It would be nicer to have the components receive the data they need in the form they need as props, rather than always transforming certain props that they receive. 

## Using Reselect (Redux’s Preferred Method to Derive Data)
Redux’s [official documentation](https://redux.js.org/recipes/computing-derived-data) recommends the use of [Reselect](https://github.com/reactjs/reselect) to compute derived data from the store. Reselect uses simple getter selectors to access slices of the state, and then performs transformations on those slices of the state to return the data in the form you want. The crucial part is that Reselect caches the return value of the selector, so that it only recalculates when the slices of the state that the selector depends on change. If none of them changes, then it just returns the cached value. The above `getPostedTransactions` selector would look like this using Reselect:

```javascript
const getPostedTransactions = createSelector(
  [getTransactions], 
  transactions => 
    _.filter(transactions, txn => txn.status === 'POSTED')
);
```

`getTransactions` returns the transactions slice of the state, which is then passed as an argument to the function that’s the second argument to `createSelector`. We can then use `getPostedTransactions` in our `mapStateToProps` function in the component.

If we wanted to use multiple selectors that return different slices of the store, we could also do that in the following way:

```javascript
const getPostedTransactions = createSelector(
  [getTransactions, getTransactionsFeatureFlag], 
  (transactions, flag) => _.filter(
    transactions, 
    txn => txn.status === 'POSTED' && flag.value
  )
);
```

You can also access props from inside the getter selectors if we want to select a slice of the state based on a prop by doing something like this:

```javascript
const getTransactionsForCategory = (state, props) => 
  state.transactions[props.category];
```

```javascript
const getPostedTransactionsForCategory = createSelector(
  [getTransactionsForCategory],
  transactions => _.filter(
    transactions, 
    txn => txn.status === 'POSTED'
  )
);
```


However, **there is an important caveat!** If a Reselect selector is used in multiple instances of components and relies on props, it will **not** correctly memoize. Each separate component will be using the same exact selector which only memoizes once. Therefore if different components use `getPostedTransactionsForCategory`, with different `props.category` values, it will not memoize properly. This is because `getTransactionsForCategory`, will return a different slice of the state each time `getPostedTransactionsForCategory` is invoked, and so the `transactions` argument to the function that does the transform changes. The arguments to the transform function govern the memoization, and so we will re-run the transform function, which then returns a new array to `mapStateToProps`, every single time. 

### Sharing Reselect Selectors Across Multiple Components
If a selector needs to be used across multiple components or multiple instances of a component, in order to memoize properly each instance of the component needs access to its own private copy of the selector. We then do something like this:

```javascript
const makeGetPostedTransactionsForCategory = () => createSelector(
  [getTransactionsForCategory], 
  transactions => _.filter(
    transactions, 
    txn => txn.status === 'POSTED'
  )
);
```

Then, the Redux `connect` function can accept a function that returns a function for its first argument (the one corresponding to `mapStateToProps`). If the `mapStateToProps` argument passed to `connect` returns a function instead of an object, it will be used to create an individual `mapStateToProps` function for each instance of the component.

```javascript
const makeMapStateToProps = () => {
  const getPostedTransactionsForCategory = 
    makeGetPostedTransactionsForCategory();
  const mapStateToProps = (state, props) => ({
    postedTransactions: getPostedTransactionsForCategory(
      state, 
      props
    ),
  });
};

export default connect(makeMapStateToProps)(MyComponent);
```

One note I have realized on further investigation is that this `makeGet` style needs to be used when the getter selectors rely on props. However, if the getter selectors simply rely on state, there is no need for this, as the state object is the same for all components.

Please each out to me if you have further questions!

**NOTE: Always use a profiler to check your assumptions about performance!** 



