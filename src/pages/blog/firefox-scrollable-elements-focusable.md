---
layout: "../../layouts/BlogPost.astro"
title: "Firefox scrollable elements are focusable"
description: "An attempt to explain why we have so many JavaScript build tools now, and what problems they help solve."
pubDate: "Sep 3 2022"
tags: ["a11y"]
---

<script async src="https://cpwebassets.codepen.io/assets/embed/ei.js"></script>

While I was working on a scrollable table of contents component at work (the same one that I was trying to <a href="aligning-list-items-vertically-into-columns">align list items for</a>), our accessibility consultant reported that he was experiencing a mystery focusable element between the button that toggled the table of contents and the first item inside of the TOC. The curious part was that it only happened in Firefox.

I asked him to share his screen while using NVDA in Firefox. And there it was, a blue focus indicator around… the scrollable container `<div>`?

As it turns out, **all scrollable elements are focusable in Firefox, even without a `tabindex` attribute.**

This was certainly a surprise to me, and apparently [to other folks as well](https://bugzilla.mozilla.org/show_bug.cgi?id=1069739).

## Why does Firefox do this?

I do appreciate the rationale for making scrollable containers focusable–it’s because if the scrollable element has no focusable elements inside, then it’s impossible to scroll the element downward without using a mouse:

> > This behavior may be beneficial, when the scrollable element contains no focusable children. If the scrollable element would not be able to receive focus, then a keyboard-only user would not be able to focus and scroll the element. This is in fact the case in Chrome, where in the test case above you can not focus the .focusable div.

> Indeed, and that is precisely why this behaviour was implemented (a long time ago).

The example below illustrates the problem: there’s no way to scroll the element downward with a keyboard.

<p class="codepen" data-height="300" data-default-tab="result" data-slug-hash="PoRrmOR" data-user="iansjk-the-decoder" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;">
  <span>See the Pen <a href="https://codepen.io/iansjk-the-decoder/pen/PoRrmOR">
  Scrollable &lt;div&gt; demo</a> by Ian Kim (<a href="https://codepen.io/iansjk-the-decoder">@iansjk-the-decoder</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>

… But if the scrollable element _does_ have a focusable element inside, then you can press Tab until you focus the element, then use the arrow keys to scroll the container up and down.

<p class="codepen" data-height="300" data-default-tab="result" data-slug-hash="ExEBvVp" data-user="iansjk-the-decoder" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;">
  <span>See the Pen <a href="https://codepen.io/iansjk-the-decoder/pen/ExEBvVp">
  Scrollable &lt;div&gt; demo with focusable item inside</a> by Ian Kim (<a href="https://codepen.io/iansjk-the-decoder">@iansjk-the-decoder</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>

That’s the confusing part of this behavior (to me): the scrollable element **remains focusable, even if it contains another focusable element**.

In our specific case, since this was a table of contents, there were a large number of links and collapsible chapter lists inside. So there would always be at least one focusable element within the scrollable container. (It wouldn’t be a very useful table of contents if it didn’t have any links!)

## How to prevent it

So, to disable this behavior requires **two** things:

1. Set `tabindex="-1"` explicitly on the scrolling element to remove it from the tab flow.
2. Add a `mousedown` listener on the scrolling element that calls `event.preventDefault()`:

   ```jsx
   scrollingElement.addEventListener("mousedown", (e) => {
     e.preventDefault();
   }
   ```

The example below applies both steps, so you shouldn’t be able to focus the scrolling element even if you are using Firefox:

<p class="codepen" data-height="300" data-default-tab="result" data-slug-hash="BaxBJMK" data-user="iansjk-the-decoder" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;">
  <span>See the Pen <a href="https://codepen.io/iansjk-the-decoder/pen/BaxBJMK">
  Scrollable &lt;div&gt; demo w/ focus forwarding</a> by Ian Kim (<a href="https://codepen.io/iansjk-the-decoder">@iansjk-the-decoder</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>

## Why the second step?

The mousedown listener is required because adding `tabindex="-1"` allows the element to be focusable by left clicking it. Try left clicking the scrolling element in the example below: you’ll see that it receives a focus indicator.

<p class="codepen" data-height="300" data-default-tab="result" data-slug-hash="abGbebB" data-user="iansjk-the-decoder" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;">
  <span>See the Pen <a href="https://codepen.io/iansjk-the-decoder/pen/abGbebB">
  Scrollable &lt;div&gt; no longer focusable in Firefox</a> by Ian Kim (<a href="https://codepen.io/iansjk-the-decoder">@iansjk-the-decoder</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>

## Caveats of this workaround

There is one important caveat with this workaround, mentioned in the bug report linked at the beginning of this article:

> Add a mousedown listener which calls event.preventDefault(). I realise this might not be feasible in some cases due to other side effects (e.g. it will stop drag events from firing), but I thought it worth noting as a potential solution.

In our use case, we didn’t have any drag events or other mousedown events on the scrollable container, so part (2) wasn’t an issue for us. Though, I’m not sure if this focusable scrolling element behavior occurs in Android Firefox + TalkBack. (Since Firefox on iOS uses WebKit rather than Gecko, this shouldn’t be an issue in iOS Firefox + VoiceOver.)
