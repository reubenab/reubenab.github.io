---
layout: post
title: "React Native's zIndex Android Bug (RN 0.45.1-0.48.0)"
date: 2017-09-08
---

Two weeks ago, while performing our usual regression testing before a release, our team noticed that our error toasts were no longer showing up in our most recent Android build, yet they were properly displaying on iOS.

## The Problem
After some digging, I noticed that the render method for the toast was executing, but the toast was just not displaying on the screen. After some further debugging, where I removed the styling and just tried to see the error on the screen, it worked. I discovered that the issue began when trying to use `position: 'absolute'` and the `zIndex` properties in styling to get the error toast to display over any other view in the same place. A quick google search led me to [this GitHub issue](https://github.com/facebook/react-native/issues/8968#issuecomment-291762006). 

It turns out that in React Native 0.45, a bug was introduced in Android that prevented it from dynamically re-rendering views based on `zIndex`, and so essentially in dynamic situations, (like conditional renderings), the `zIndex` property was ignored by Android. We had recently upgraded from 0.44 to 0.45 to use some auto scroll functionality that was added to `SectionList`, but this seemed to have broken our error messages on Android.

## The Fix
One possible solution may have been to use the `elevation` property on Android, instead of `zIndex`, but that will not work for all cases when `zIndex` is needed, since according to [this comment](https://github.com/facebook/react-native/issues/8968#issuecomment-316225419) on the same thread, pointer events pass through what is underneath with `elevation`, unlike `zIndex`, and `elevation` also casts a shadow.
Fortunately, the fix that worked for us in this case turned out to be very straightforward. We already had a wrapper component around every screen in the app that had a render method that looked something like this:
```javascript
render() {
  const { children, showError } = this.props;
  return (
    <Wrapper>
      {showError && <ErrorToast />}
      {children}
    </Wrapper>
  );
}
```
Our fix simply consisted of swapping two lines in this file, so that the render method looked like this:
```javascript
render() {
  const { children, showError } = this.props;
  return (
    <Wrapper>
      {children}
      {showError && <ErrorToast />}
    </Wrapper>
  );
}
```
This works because rendering in Android happens component by component. Therefore, the `ErrorToast` component now renders after all the other components in a screen have been rendered, rather than before it, and so with absolute positioning, it is rendered on top of the other components, instead of having other components rendered on top of it. When the conditional changes, the renderer simply then renders this component on top of the other components on the screen.

## How to Identify the Issue Sooner
We have had other issues related to this bug and how we use other components, which unfortunately do not have as simple fixes. Ideally, we would have discovered this bug as soon as we upgraded to React Native 0.45, and so would have had adequate time to come up with good long-term solutions, rather than a quick patch so that we can meet our release schedule. 

This issue was fixed as of React Native 0.48.2, and so perhaps we could have explored upgrading all the way, but this is too risky to do right before a release. I personally do not spend as much time testing on Android as I should, since typically when I am building new features, the iOS simulator is faster and provides a better development experience, and also I typically build for the "happy path" initially, and since React Native is iOS first, there tends to be fewer bugs in iOS. 

As a general practice, I have decided to thoroughly test on Android every time I am happy with a part of the feature on iOS. This may slow my development cycle slightly on a day-to-day basis, but hopefully it will speed up my process overall, since I will spend less time having to devise roundabout solutions to tricky problems discovered right before a deadline. 

