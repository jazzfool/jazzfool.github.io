---
title: "The grim GUI landscape"
date: 2021-07-31T22:22:55+10:00
draft: false
---

I'd like to think I've spent *long enough* in desktop GUI development. I'm disappointed by what I've seen as I've worked with a vast palette of desktop GUI solutions. Everything from Qt to Electron have their own hideous limbs with their very own creative ways burden you as you develop your app. That isn't to discredit them all, as a few actually offer wonderful developer experiences (though often still lacking in practicality and applicability).

## Electron

Electron is perhaps the most poor desktop GUI solution to pop up and it seems to have bred rapid adoption of browser technology into every corner of computing. The main problem with dragging along an entire browser isn't that it's bloated or memory-consuming (albeit those are still problems). It's that they include a plethora of hidden mechanisms that will never benefit a fully controlled web-based environment (i.e., your app's webpages). Think of the thousands of security patches and workarounds, the intricate DOM optimizations, the multi-webpage inter-process resource sharing. None of that will benefit an Electron app and will only slow it down. An Electron app will only display 3-4 webpages, all of which are fully controlled by the developer. Browsers, on the other hand, are designed to handle incredibly rare edge cases and exploits in the wildest parts of the internet.

To put it shortly, you're misusing the software. It's astonishing how some people continue to defend the use of Electron for its "developer productivity". There is a hint of truth in that, but not in the way they think. Their defence is greatly misplaced. It's not that Electron or Chromium itself gives any developer productivity; it's the web ecosystem and front-end frameworks.

For all that I bash Electron, I believe it has taught a couple important lessons:

1. There's a huge appetite for a *declarative* desktop GUI library. 
2. Developers want support from an ecosystem, especially for a GUI.

## Qt

Qt is a passable desktop GUI solution, but it still has its problems.

Firstly, your application lives *inside* Qt, instead of the other way around. Instead of Qt being a library you can plug up to your own code so you can throw on a GUI, you must immerse your app completely and utterly, typically by developing with the Qt library from the very start. This means there are many clear boundaries in your code where, for example, Qt and stdlib meet (say converting an `std::vector` from another library into a `QList`).

Moreover, I find imperative GUI libraries, such as Qt, quite unsatisfying in terms of expressiveness, readability, and cleanliness. There's no clear flow of data through your GUI, as if you were to map out all the callbacks and subsequent state mutations, it would practically be spaghetti. This is in contrast to something like React where the data flow is strongly driven in a single direction. Here also lies a major architectural issue: There is no single source of truth. This occurs even at the smallest of scales, for example when syncing a label's text with your data.

Styling and other visual parts of Qt also leave a lot to be desired. I'm shocked that Qt does not default to GPU accelerated widget rendering when available. Screen resolutions are increasing and CPU rasterization simply cannot keep up.

Overall however, it gets the job done. It manages to stay relatively low (compared to Electron) on memory consumption while offering a broad range of utilities and features. There is still much room for improvement though.

## WPF (and other Microsoft UI)

WPF and its ilk are a great follow up to Qt, as they solve the problem of lacking a single source of truth and integrate seamlessly with the rest of C#.

XAML offers property binding which is an acceptable way to solve the problem of synchronising two stores of data. By simply binding to a variable when setting the property of a widget, you can trivially bind two different datum and never have to worry about it again. On the other hand, having to manage the gap between C# and XAML is still an issue. Things start to get messy when trying to mutate the widget tree on the C# side, and vice versa. Amongst the community there seems to be a majority consensus of using MVC (or some other framework such as ReactiveUI) to architect your application around. Beyond my personal disgust for MVC (see why [here](http://web.archive.org/web/20210728123238/https://acko.net/blog/model-view-catharsis/)), it feels rather restricting that you're expected to lock yourself into a very specific design.

Finally, of course, there is the problem that it is exclusive to Windows. There is Avalonia, which is a step in the right direction, but it is still too young to see major adoption and support. Another point of concern is that Microsoft seems rather unstable and trigger-happy in terms of their UI story.

## Flutter

Flutter began as a mobile UI library and it is very good at that. Meanwhile, the desktop port seems unfinished, neglected, and mostly just appears to be an afterthought.

It is very clear that Flutter is highly focused on mobile platforms given the oversized widgets, sliver headers, bottom swipeable tabs, flick to scroll, hero animations, and other widget features you would never find in a desktop-focused GUI. This is quite unfortunate since the Flutter architecture itself is terrific thanks to an attractive declarative API backed by an accessible retained element tree.

This by all means can be fixed by building a separate widget library designed specifically for the desktop from the ground up. Somehow though, I doubt this is a venture Google would want to fund. It's a shame since Flutter's architecture shows so much promise in terms of ease of access, productivity, and escape hatches.

## Immediate-mode

Dear ImGui (and other immediate-mode UI) present a very elegant solution to many of the problems I've mentioned. For one, you don't need to bother with MVC or any kind of binding or callbacks since the very notion of immediate-mode UI revolves around storing and pulling to/from a common data source. Moreover, Dear ImGui doesn't ask for a large widget interface to implemented or for custom data structures; you can reuse your existing data structure to describe a UI. It is very much plug-and-play in that regard and feels like an integratable library rather than a bloated framework.

Even issues that some cite as being downsides of Dear ImGui are fixable. You don't have to render at 60 fps - only render when you receive an event. You don't have to regenerate the entire tree every frame - perform some diffing to reuse and recycle computation. Most of these supposed issues boil down to the implementation, not with the architecture itself.

With that said, there are still some *fundamental* downsides with being fully immediate-mode. One such downside is layout. Since everything is computed in one go, you can't backtrack to fix up layout after receiving new information (e.g., sizing to fill but later discovering more room to fill; too bad, you can't go back). At this point, you'll have to resort to storing an intermediate tree to do a separate layout pass.

If you're willing to give up some of the purity of full immediate-mode and begin to introduce intermediate trees and layout callbacks then you could very well fix these issues. Though, I also find there is a mentality in immediate-mode UI that it should only be used for internal tooling (mostly for game engines). I whole-heartedly disagree. If you choose to make it more robust with intermediate trees and whatnot, then you have a React-esque library on your hands which is more than capable for more complex desktop applications, only missing a desktop focused widget library (as opposed to tooling focused).

## Closing thoughts

There are many GUI libraries I left out, so I'll briefly explain why:

- FLTK, Tk, libui, Lazarus, etc: They are not as robust as one would wish for more complex applications. They do not offer many escape hatches as they are really an abstraction of the OS GUI (which also do not offer many escape hatches). As a result, things such as custom widgets and styling are greatly lacking.
- JavaFX, Swing, GTK, etc: These generally fall in the same camp as Qt and WPF.

GUI is an incredibly difficult problem and an incredibly ugly problem. There is no elegance to be found here. A good GUI API implementation is one that is willing to be ugly for the sake of performance, accessibility, and usability.

