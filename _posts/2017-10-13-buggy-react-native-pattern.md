---
layout: post
title: "The JS Pattern That Could Break Your React Native App"
date: 2017-09-08
---

## Raw text must be wrapped in explicit <Text> component

Last week, our app suddenly started crashing for all our mobile developers on stage with the above error. The fun part - there was no commit that this could be immediately traced to. Even after rolling back to commits from days ago, the app continued to crash, despite the fact that at that time those commits were made, it was running seamlessly.

## The Error Message
The specific error message that displayed was that we had a raw string that was not surrounded by a `<Text>` component, and so the app was unable to properly render it and threw an error. Unfortunately, the error did not display specifically which line or file this was happening on. None of us could continue developing and testing while this error persisted, so our whole team started investigating the issue. 

We also realized that since no specific commit seemed to have caused this issue on our end, the problem was likely surfacing due to a change from the backend, so we started looking at the results of our API calls on the screen that was crashing.

## The Root Problem
We noticed that there was a new object in the array that our backend was returning, and one of the fields had an empty string, which had not yet ever been sent (previously the field was either null or had a value). We then looked at how we were rendering the fields from that object and found where the empty string was used. This is what the `render()` function looked like:
```javascript
render() {
  const { x } = this.props;
  return (
    <View>
      {x && <SomeOtherComponent text={x} />}
    </View>
  );
}
```

In most cases, this is totally fine. If `x` is null or undefined, that whole line does not render. If `x` is some non-empty string, `<SomeOtherComponent />` is correctly rendered. However, in the case that `x === ''`, the `{x && <SomeOtherComponent text={x} />}` expression evaluates to `{''}` since the empty string is falsey, meaning that `<SomeOtherComponent>` does not get rendered. However, since the empty string is still of `String` type, React Native sees that it is not surrounded by a `<Text>` component and throws an error. 

## Potential Solutions
There are a few different ways that we could write a solution to fix the error.

* __Solution 1:__ `{!!x && <SomeOtherComponent text={x} />}` - the issue with this is that it is not necessarily a generalizable pattern for an empty object or empty array if you do not want to render `<SomeOtherComponent>` in those cases.
* __Solution 2:__ `{x ? <SomeOtherComponent text={x} /> : null}`
* __Solution 3:__ `{!_.isEmpty(x) && <SomeOtherComponent text={x} />}`

In my opinion, React Native should handle `{''}` the same way it handles `{null}` and `{false}`, but apparently the existing functionality is working as intended, as seen by [this Github issue](https://github.com/facebook/react-native/issues/15870) that was closed.

## Key Takeaways
* Test edge cases more thoroughly - specifically using `null`, `undefined`, and the falsey value.
* Be defensive and explicit with your conditions
