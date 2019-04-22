---
description: >-
  This is a walkthrough guidebook for anyone who has faced the "aparently easy"
  issue of migrating a Typescript (or even pure JS) legacy project from
  namespaces to a ES modules architecture.
---

# Typescript: Migration from Namespaces to Modules

## Prologue

If you are reading this I suppose you are part of these few “legacy” projects done in Typescript \(probably migrating from an initial JS codebase\) where you compile into a single JS file or you use “script tags” for every compiled JS file.

If you are one of those lucky spoiled front-end developers starting with a fresh “create-react/angular/vue-app” which does not need to reuse any code from a legacy app, then you have not faced the problem described in this article and you are better that way \(take my word\).

These kind of legacy projects we focus on work in the “global scope”, but of course, to better structure your code and provide encapsulation you had been using Namespaces, which are great! and even are able to mix and provide encapsulation across different files.

Now, you have noticed things have changed for better in the front-end ecosystem and the standard for structuring your code in JS are modules. All the building tools today \(like webpack\) understand modules and are able to provide some capabilities when you use them like: code splitting, lazy loading of code, inversion of control, tree shaking, clear dependencies... just to name a few.

But of course, if we are talking about those legacy projects you are not in this league, you have been forgotten forever and if you want to benefit from any of those capabilities you are going to have to implement them all by yourself, which would be even crazier than the path we will explain in this article: **migration to modules**.

In fact you will face this challenge in two scenarios, as far as I am concerned:

* You want to modernize your project and benefit from all the new standards and tooling around modules \(which I have commented before\)
* Or \(as it was my case\), you want to **reuse code** which exists in your legacy app from your “brand new create-xxx-app”. \(or even the other way round\)

In any of these cases you will need to follow the steps described here.

{% hint style="info" %}
Note: If you are not working with Typescript but you still are using the global scope \(not modules\) in your codebase, this guide will be also useful for you. Your problem is just a subset of the problems solved here.
{% endhint %}



