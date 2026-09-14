# Explainer for the Unwanted Work API

This proposal is an early design sketch by fergal@chromium.org to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- fergal@chromium.org

## Participate
- https://github.com/explainers-by-googlers/work-vs-idle/issues

## Table of Contents [if the explainer is longer than one printed page]

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [User research](#user-research)
- [Use cases](#use-cases)
  - [Use case 1](#use-case-1)
  - [Use case 2](#use-case-2)
- [[Potential Solution]](#potential-solution)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
    - [Use case 1](#use-case-1-1)
    - [Use case 2](#use-case-2-1)
- [Detailed design discussion](#detailed-design-discussion)
  - [[Tricky design choice #1]](#tricky-design-choice-1)
  - [[Tricky design choice 2]](#tricky-design-choice-2)
- [Considered alternatives](#considered-alternatives)
  - [[Alternative 1]](#alternative-1)
  - [[Alternative 2]](#alternative-2)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Sites often unintentionally keep the CPU awake or even heavily loaded
while doing nothing useful.
This can be due to bugs in JS,
animations that are expensive
or that are running even nothing is visibly animating.

This can result in battery drain, hot laptops
and slow performance of the site itself and other sites.

## Goals

Allow sites to
- quantify time spent doing work vs idle
- identify the root causes of unintended work
- distinguish work by common categories, JS, animation, media, etc.
- distinguish intended work from unintended work
   - it's not a bug for a movie player to being constantly playing a movie
   - it's not a bug for a spinner to spin
     *while* the page is visible and waiting for something to complete

This API should be low-enough overhead to be always-on
so that unintended work can be discovered
- early in development and internal dogfood usage
- in the wild

This API is intended to be a tool
for sites to achieve long periods of idleness
when the user is not actively engaged with them.

## Non-goals

This is not for
- measuring CPU usage
- measuring battery usage/level
- profiling

## Use cases

There are multiple well-known sites
that I regularly kill from the browser's task manager
because my laptop is hot, my fans are spinning
and those sites are using constant CPU
indicating that something is happening maybe with every frame
while doing nothing visible or useful to me as a user.

### Longitudinal monitoring

Over time, a site measures
- its work vs idleness
- its time-to-idleness after user interaction

It sees a sudden regression in a new version.
The devs use data from the wild to identify the cause.

We know that some sites have attempted to measure
time-to-idleness using LoAF.

### Immediate feedback to devs

For devs,
provide a visual indicator
when a page has been unintentionally non-idle
for a significant part of the last 10-20s.
This could be a library dropped into every page.

### Detect expensive or unintended animations

We know that teams at Google and elsewhere
have already developed "bad animation" detectors
based on LoAF and other APIs.
These have drawbacks like requiring actual dropped frames
(which often don't happen on the high end machines that developers typically use)
and intrusive code changes or monkey-patching.

<!-- In your initial explainer, you shouldn't be attached or appear attached to any of the potential
solutions you describe below this. -->

## More info

For now this repo and explainer is a place-holder.
An API is described in this [slide deck](https://docs.google.com/presentation/d/1d8VaGHF9OFF9Kuy--jIUvYLcGouPtJg7Yxtryk3HvMc/edit).
This API was discussed at the [WebPerfWG meeting on 2026-09-10](https://docs.google.com/document/d/10dz_7QM5XCNsGeI63R864lF9gFqlqQD37B4q8Q46LMM/edit?tab=t.0#heading=h.pndss1ey0460)
and this repo has been created to facilitate discussion.
The content from that slide deck will be moved into this explainer.

There are many issues with the API shape of this proposal
- maybe it should align with `PerformanceObserver`
- maybe it should align with the JS Profiling API

Right now, the API shape is secondary to figuring out
- what would actually be useful (and used in reality)
- what should be part of the API and what should be left to be implemented in JS around the API

# This explainer is incomplete

*The document from here down is the remaining parts of the template.
It will be filled out soon.*

[For each related element of the proposed solution - be it an additional JS method, a new object, a new element, a new concept etc., create a section which briefly describes it.]

```js
// Provide example code - not IDL - demonstrating the design of the feature.

// If this API can be used on its own to address a user need,
// link it back to one of the scenarios in the goals section.

// If you need to show how to get the feature set up
// (initialized, or using permissions, etc.), include that too.
```

[Where necessary, provide links to longer explanations of the relevant pre-existing concepts and API.
If there is no suitable external documentation, you might like to provide supplementary information as an appendix in this document, and provide an internal link where appropriate.]

[If this is already specced, link to the relevant section of the spec.]

[If spec work is in progress, link to the PR or draft of the spec.]

[If you have more potential solutions in mind, add ## Potential Solution 2, 3, etc. sections.]

### How this solution would solve the use cases

[If there are a suite of interacting APIs, show how they work together to solve the use cases described.]

#### Use case 1

[Description of the end-user scenario]

```js
// Sample code demonstrating how to use these APIs to address that scenario.
```

#### Use case 2

[etc.]

## Detailed design discussion

### [Tricky design choice #1]

[Talk through the tradeoffs in coming to the specific design point you want to make.]

```js
// Illustrated with example code.
```

[This may be an open question,
in which case you should link to any active discussion threads.]

### [Tricky design choice 2]

[etc.]

## Considered alternatives

[This should include as many alternatives as you can,
from high level architectural decisions down to alternative naming choices.]

### [Alternative 1]

[Describe an alternative which was considered,
and why you decided against it.]

### [Alternative 2]

[etc.]

## Security and Privacy Considerations

[Describe any interesting answers you give to the [Security and Privacy Self-Review
Questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/) and any interesting ways that
your feature interacts with [Chromium's Web Platform Security
Guidelines](https://chromium.googlesource.com/chromium/src/+/master/docs/security/web-platform-security-guidelines.md).]

## Stakeholder Feedback / Opposition

[Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]

## References & acknowledgements

[Your design will change and be informed by many people; acknowledge them in an ongoing way! It helps build community and, as we only get by through the contributions of many, is only fair.]

[Unless you have a specific reason not to, these should be in alphabetical order.]

Many thanks for valuable feedback and advice from:

- [Person 1]
- [Person 2]
- [etc.]
