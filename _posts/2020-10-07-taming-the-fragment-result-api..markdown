---
layout: post
title:  "Taming the Fragment Result API"
date:   2020-10-07 10:00:00 +0300
categories: androidx kotlin
---
_Disclaimer: This article is written by a developer who have not yet migrated their fragments to Jetpack Navigation Component, therefore lack crucial experience in it. Although the author have researched the use-cases of Fragment Result within Navigation Component, some of the things that are discussed below may not hold in some unforeseen situations._

Fragment Result API was introduced in Fragment artifact of AndroidX with the version `1.3.0-alpha04` . Since then, it hasn’t received many changes to either its API or implementation detail. Its main goal was noted in the release notes:

> Added support for passing results between two Fragments via new APIs on `FragmentManager`. This works for hierarchy fragments (parent/child), DialogFragments, and fragments in Navigation and ensures that results are only sent to your Fragment while it is at least `STARTED`

At the same time another method for passing data between fragments was deprecated.

> The target fragment APIs have been deprecated. To pass data between fragments the new Fragment Result APIs should be used instead

These changes were quite a relief because there were multiple options for passing data between fragments at the time and none of them felt like the right thing to do for their own reasons. Let’s start by briefly going over them and understand why FragmentResult was a necessity.

## The old ways

### Target Fragment

`TargetFragment` had been the recommended way of passing a result from a fragment to parent fragment. Its API leveraged the already existing `onActivityResult` method in fragments. Simply, a fragment could return its result in a way that is very similar to Activities. To learn more about it you can check out [this post](https://medium.com/better-programming/what-is-target-fragment-da0e7c7f345c).

There wasn’t a particular problem with `targetFragment` until Navigation Component had arrived. When parent fragment wanted to start e.g., a dialog fragment which would return a selected item, it simply called `setTargetFragment` and child fragment could easily return the result to parent’s `onActivityResult` . It was just a convenient way to pass the parent fragment information to a child fragment. However, Navigation Component handles fragment creation including arguments in which there is no control over setting the target fragment.

Recommended way of handling this problem at the beginning was to use of shared ViewModels. However, using shared ViewModels to just deliver result data between a child and parent could be called an anti-pattern. It requires a single ViewModel that governs both views at the same time. This approach could get infeasible very fast in the case of merging two very different ViewModels just to achieve a data exchange.

Therefore, `targetFragment` was deprecated at the same time FragmentResult was introduced.

### Callbacks

There is not much to discuss about callbacks. They are comfortable, favorable, easy to follow and adheres to traditional way of assessing results from a callee to a caller. However, they come with a HUUUGE drawback. Once the activity is destroyed for some reason and recreated, the fragment backstack is also reinitialized with initial arguments as well as what was saved during `onSaveInstanceState` phase. Unfortunately, callback lambdas are lost forever. There is no lambda implementation that can be put in a bundle.

{% gist 921245c0c50e846c63030ada4fa58eba %}

### Activity Interfaces

Introducing an interface in the activity that can receive the result from a fragment is the oldest but the safest way. But again, this doesn’t really cover fragment to fragment scenario which is our main goal with the Navigation Component. In a SingleActivity world, almost all communication is going to happen between fragments. Although they are still very useful, activity interfaces don’t help us in this quest. On a side note, it wouldn’t hurt to find a solution that can also easily replace Activity interfaces so that we have a unified solution for both _fragment_ to _activity_ and _fragment_ to _fragment_.

## FragmentResult is … verbose?

I can attempt to explain the inner workings of FragmentResult and how to use it properly but I would definitely butcher it. So if you have questions about it and you’re not in a hurry, I again would like to suggest reading the official documentation and the post I shared below.

[Pass data between fragments](https://developer.android.com/training/basics/fragments/pass-data-between)

[Android Fragments: Fragment Result](https://proandroiddev.com/android-fragments-fragment-result-805a6b2522ea)

Let’s take a look at the examples from the documentation. In the end, we might have some questions regarding the ease of using this API.

{% gist 8e213650f50f9dacd41814e8d938cb41 %}

This is the example given in official docs for the API. Although `fragment-ktx` is optional, I definitely recommend including it in your dependencies because it properly takes advantage of Kotlin.

What we see in this example is the sample code for how parent fragment is able to listen to the result that is provided by a child or sibling fragment. Fragment-unawareness is the first thing that catches our attention. We are not setting something like `targetFragment` to the child fragment nor we are defining it in this listener setup. All that is expected is a request key and a bundle key. Let’s keep that in mind for future reference.

{% gist a5b4c27782403c82f8847052733f1a83 %}

Above code is the example for the fragment that wants to return a result. Once again, it doesn’t care about the type of the parent or actually which parent we are passing this result to. Only configurables that are apparent in this example are request key and bundle key.

### What about the keys?

It’s all cool and nice that these examples demonstrate the fact that Fragment Result is approachable and can replace the older methods quickly. However, there is something that is grossly overlooked in the documentation: `"requestKey"` , `"bundleKey"` . Would you be OK with having constant values defined in-place? In most circumstances, these keys would be better defined as constant values in companion objects.

```
private const val REQUEST_KEY = "requestKey"
private const val BUNDLE_KEY = "bundleKey"
```

Are we done? Not really. Let’s talk about a scenario where you may call a dialog fragment for different results. For example, parent fragment needs to receive a start date and an end date from a child fragment. You need to differentiate between the two results coming for both start and end date. How would you do that? Easy, just use different request keys. This introduces a new kind of problem; you need to make sure the child fragment knows which request key you are using. Easy, just pass the request key through the arguments. Wait, hold up! This added a little bit verbosity on both ends.

{% gist 39c513d587b53554e2b995a2fa8165d7 %}

Both the parent fragment and child fragment needs to figure out which request key to use. We haven’t even talked about bundle or result key that is going to be the key for result in the bundle. That also needs to be added as a constant value in either child or parent fragment.

In the above example, we also assumed that the arguments were non-existent to begin with. What if the child fragment already had a factory function such as `newInstance()` ? Wouldn’t it be risky to make public interface changes so that we can have the advantages of FragmentResult?

Before moving forward with introducing a plain approach to get rid of this verbosity, let’s make assumptions on general use cases.

1. Most fragments return a single result. We can think of them as functions. Even when they have multiple values to return, they can be wrapped in a bundle. In the end, result can be anything that is `Parcelable`
2. Request Keys are not constant. This can still be configurable.
3. Most common use-case is one-to-one where an activity or a fragment calls a child fragment only for a single objective. This default behavior should be achieved very easily.

## Rizalt

I have to admit that there is no alternative to `setFragmentResult` and `setFragmentResultListener` when it comes to naming. Adding helper functions that also have the same name with similar signature would be quite confusing. That’s why I had to choose a different name just for the sake of it. `Rizalt` is the Turkish spelling of how `Result` is pronounced in English.

I will give you the full implementation of `Rizalt` down below but let’s first talk about what kind of methods that we are going to need and why?

Starting with; we are going to need to mark the child fragment that it is intended to have a return value. This way, all our helper functions can be written as extensions on that type.

Secondly, it is necessary to create a customizable request key that also has a default value when it’s not cared for.

Third, result key should also be inferred by looking at the child fragment and doesn’t need customization. Our first assumption was that the result would be a single return value.

Finally, alternatives for `setFragmentResult` and `setFragmentResultListener` which are all aware of these keys that were created and passed in the background.

So, here we have the implementation for `Rizalt` .

{% gist c9f97ad817bd4de168e5f597a6fcfe00 %}

And example usage

{% gist 50d6aacbc2e6a88b7ab238d7be4948e4 %}

The idea is as follows;

- As long as you have no need for a custom request key, it can be omitted. Default request key will always be calculated.
- When custom request key is required, it can be used in result listener for different requests but it is not really needed when registering the result from child. At that point the request key can be either read from arguments or again calculated by default value.
- `setFragmentResultListener` now knows about the fragment class, rather than a String. It’s more natural to me that result listener actually knows which fragment it is listening to instead of using an arbitrary string.
- Whatever can be inferred, should be inferred. Result listeners now receive the exact return type that is defined in Rizalt interface, rather than a bundle.
- `setFragmentResult` was simple and OK, so the API for it is not modified. However, the new function also supports the default bundle key.

## Conclusion

This write-up turned out to be longer than I intended at the beginning but I wanted to be thorough so that what is introduced is adequately justified. FragmentResult is awesome but it required adding a bit verbosity to our current codebase which didn’t feel right. This approach also introduced compile-time type check and inference which would be lost by passing bundles between fragments and then casting Parcelables to intended types.