---
title: Setup Wizard Routing -- Browser Back
fileName: setup wizard routing -- browser back
tags:
- tag
categories:
- category
date: 
lastMod: 
---
# `SetupWizardIndex`

  + `synchronizeRouteAndMachine`
    + Called `(xstate) onTransition`

    + Called during `created()`

    + `state` within this method refers to "the machine at a moment in time"

    + Initializes `Lockr...savedState` if it exists

    + The `vue-router` shaped `meta.route` property from the XState "state node" definition is pushed to the router.

  + `Lockr...savedState`

    + Is updated `(xstate) onTransition` with the machine's current state

    + Is read whenever the page is `created`

# Handling Browser Back

  + `window.addEventListener("*", ...)` is not really ideal for our purposes. There is no way to know upfront if the user is going forward or backward.

  + `vue-router` navigation guards can be used as a sort of middleware for tracking, reacting to and modifying however the user is attempting to navigate

# The expected flow

  + ## User initiates change of route via buttons in UI
    + `wizardService` receives an event and transitions (or doesn't) accordingly

    + The `onTransition` handlers are fired off, resulting in:

      + `Lockr...savedState` being updated

      + `synchronizeRouteAndMachine`
 is called (and the user is rerouted)

  + ## User initiates change of route via browser actions (Forward **OR** Back)

    + `vue-router`moves the user to another `route`

      + This results in none of the transition business occurring as in the ## User initiates change of route via buttons in UI
 flow

    + ### Idea(s)

      + #### Inject the `wizardService` and logic into `$route.meta` on all routes

        + Start with expecting several new `$route.meta` properties:

          + 1) `wizardRouted` - When true, indicates that the user has started the navigation via the UI

          + 2) `backEvent` - The event which we will send to the state machine if the user is using browser navigation

          + 3) `noBackAction` - This overrides whether we ever send the `backEvent` or not

          + 4) `wizardService` - The shared service used within the plugin

        + Update `synchronizeRouteAndMachine`
 to set `context.wizardRouted` to `true` every time before pushing the route

        + With a `beforeRouteUpdate` guard in `SetupWizardIndex` we can ensure that `this.$route.meta.wizardService` is always set to the latest instance of `wizardService`

        + In a `beforeRouteEnter` in `OnboardingStepBase` we set `$route.meta.backEvent` and `$route.meta.noBackAction`to the value of the props of the same names given to that component

        + In a global `beforeResolve` guard in `routes.js` we will now have the information we need to know

          + Ask what `$route.meta.noBackAction` is:

            + `true` - We can just go `next(from)` from here I think as we will be preventing the navigation

            + `falsy` - We continue on

          + Ask what `$route.meta.wizardRouted` is:

            + `undefined` ... `falsy` - Then we know we're navigating via means other than the UI

              + We send the machine the `$route.meta.backEvent`

                + This should then trigger `synchronizeRouteAndMachine`
, which then triggers the proper route

                + OR We `next(to)` to ensure the updated route object is passed (without the `wizardRouted` property)

            + `true` / `truthy` - Then we know the user has clicked the UI and that this route handling has been handled elsewhere (ie, we can just let the router continue to do its thing)
