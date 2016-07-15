---
layout: post
title: "Good Code Design Makes Better Programs"
date: 2016-07-14
---

This post is about the importance of internal *code design* - although UI/UX design is also *extremely* important (nobody wants to use an app that looks and feels bad).

## What is Good Code Design?

Here are a few principles that are part of well-designed code:

* __Readability:__ Code should be easy to read and understand for other developers so that they can add to it and iterate on it.
* __Simple Structure:__ The program should follow a logical execution flow and classes should have an easy to follow, linear hierarchy. This again makes it easier for others to understand code, but it also helps avoid bugs, as we will see later.
* __DRY (Don't repeat yourself):__ Similar code in multiple places should be abstracted out into a single function. This increases development speed and makes it easier to make changes.
* __Reusability:__ Create multiple classes that can be reused (by extension or passing various parameters) for slightly different functionality. Again, this increases development speed and improves reliability.

The general rule is that code should be easy to change as requirements change (which means it should be easy to read, understand, and iterate on). For a more thorough description of good design, see [this blog post](http://www.artima.com/weblogs/viewpost.jsp?thread=331531).

## An Example

Last week, I was tasked with creating a native banner for our iOS app that slides down from the top of the screen when needed, and can be dismissed when swiping up or tapping close. There were two defined use cases: one for text alerts (like errors or success messages), and one to prompt users for feedback about their in-app experience. I was advised to make a general `Banner` object that will then contain other views, like the text view or the feedback view. So I thought about it for some time, and after realizing that this view will be created from some arbitrary controller, I decided to make a `TextBanner` object that would contain the banner as a member and then fill the banner with the required text. Any future banner objects could be created in a similar way - create a class that contains the `Banner` object, create the view that goes inside, add that view to the banner, and show the banner when needed.

## The Selector Button Click Bug

The `TextBanner` worked fine, but unfortunately the buttons on my `FeedbackBanner` were not working. For some reason the `Selector` for those buttons, defined in my `FeedbackBanner` class, was not firing - no error was thrown, the callback function just was not being called. The target for the button was `self`, so for debugging purposes, I created a function in the `Banner` class with the same name, and then button clicks were executing that function. I began to think I misunderstood how the target worked using `self`, so I tried to make sure that the target was referring to the `FeedbackBanner`, first by setting a variable to self outside the `button.addTarget()` function call and then referencing that variable, but that didn't work. So then I created a new instance of the `FeedbackBanner` and tried passing that as the target. Still no luck. Finally, I made the `Selector` function static and passed the target statically as `FeedbackBanner.self`. Surprisingly, this time the button click worked, and it printed what I expected to console. Unfortunately, this view was not a static view, and I wanted to access its instance member variables on the button click, so a static function was not really an option. At this point, I spent hours researching potential reasons why `Selector` was not working properly, but didn't find anything that helped, and so I was stuck. All of this seemed like exceptionally odd behavior (why would a static reference work, but not a non-static one?).

## The Fix

I explained the situation to my mentor, and even he didn't seem to understand what was going on. However, he did point out that ideally we wanted a Singleton controller for this banner, so that there was only one instance of it at any given time. He explained that if it is showing error messages, but a user does not close the first one, we could have multiple banners superimposed on top of each other as new errors show up, and that would mean the user would have to close multiple in a row to clear the screen again - not a very friendly UX. He also thought that my class hierarchy was a little roundabout and suggested a simpler, more linear structure - a `BannerManager` that acts as the singleton and ensures only one banner is on the screen at once; it contains a `Banner` View, and then calling various functions governs what view goes inside that banner view. For example, for the `FeedbackBanner`, `BannerManager` creates a `FeedbackBanner` View, puts it inside the `Banner`, and then displays the `Banner`.

Lucky for me, most of the code I had written was pretty modular, so this refactor only took me about an hour. And then just like *magic*, the buttons were working and calling the expected function with target `self`. In retrospect, __the real culprit here seemed to be a circular reference:__ the `FeedbackBanner` had the `Banner` as a member, which then contained the `FeedbackBanner` as a member, and so on.

## Key Takeaways

* Cleaning up and simplifying design can fix bizarre bugs
* Good design allows for faster development cycles in the future (write modular code that is easy to change!)

Next time you spend hours trying to debug something and don't seem to be making any real progress, or simply don't understand what is causing the bug, consider your class design. Sometimes, you may even save time overall by just redesigning and simplifying that design.