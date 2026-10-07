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

Run-away work is a problem on the web.
This can be due to bugs in JS,
animations that are expensive
or that are running even nothing is visibly animating.

This can result in battery drain, hot laptops
and slow performance of the site itself and other sites.

This explainer proposes and API
that would allow sites to monitor the work that they do
in an aggregate form
that would allow for reporting and debugging of these problems.

## The Problem

### For web users

There are multiple well-known sites
that I regularly kill from the browser's task manager
because my laptop is hot, my fans are spinning
and those sites are using constant CPU
indicating that something is happening maybe with every frame
while doing nothing visible or useful to me as a user.

As a user of the web,
I want sites to pay attention to their CPU usage
both in terms of total CPU usage
and frequent wake-ups.

### For web authors

Performance suffers when sites are doing unintended work.
Users kill tabs that make their laptops hot.
The existing tools are not adequate for measurement
or root-causing.
JS self-profiling is
- heavy-weight, so not suitable for continuous measurement
- produces a lot of data (full stack traces) that is not necessary for attributing initiator
- samples at an interval so can easily miss short-duration wake-ups
- only covers JS, not animation, layout etc

### For browser vendors

Browsers are blamed for using CPU.
Vendors put effort into detecting poor performance caused by sites
and prompt users to kill those sites.
There is no signal back to the sites.

## Goals

Allow sites to
- quantify time spent doing work vs idle.
- identify the root causes of unintended work.
- distinguish work by common categories, JS, animation, media, etc.
- distinguish intended work from unintended work.
   - it's not a bug for a movie player to being constantly playing a movie.
   - it's not a bug for a spinner to spin.
     *while* the page is visible and waiting for something to complete.
- distinguish between repeated wake-ups and solid CPU usage.

This API should be low-enough overhead to be always-on
so that unintended work can be discovered and debugged
- early in development and internal dogfood usage
- in the wild

This API is intended to be a tool
for sites to achieve long periods of idleness
when the user is not actively engaged with them.

## Non-goals

This is not for
- measuring battery usage/level
- code profiling
- measuring CPU usage
  (although some CPU usage stats may be produced)

## Use cases

### Longitudinal monitoring

Over time, a site measures
- its work vs idleness
- its time-to-idleness after user interaction

It sees a sudden regression in a new version.
The devs use data from the wild to identify the cause.

We know that some sites have attempted to measure
time-to-idleness using LoAF.

### Immediate feedback to devs

There could be a drop-in library
that will monitor work and idleness
and fire an event when some threshold of work is crossed.
E.g. < 80% idle for the last 10s
In dev mode the site could visually indicate this
so that devs get immediate feedback
when they introduce bad changes.

### Detect expensive or unintended animations

We know that teams at Google and elsewhere
have already developed "bad animation" detectors
based on LoAF and other APIs.
These have drawbacks like requiring actual dropped frames
(which often don't happen on the high end machines that developers typically use)
and intrusive code changes or monkey-patching.

## Challenges

### Performance

We must ensure that turn on this API
does not have a noticeable impact on performance.
When no work is occurring this API has no overheard.
Hopefully, when work is occurring,
the work done by each initiator is much more expensive
than the cost of recording that work.

### Distinguishing intentional work from unintentional work

As far as the browser is concerned,
all work is intentional.
It cannot distinguish an animation
that is running intentionally
from one that is running unintentionally.
There should be a way for devs to signal
that some work is ongoing
so that that work can be tagged.
Unfortunately, providing this kind of mechanism
opens up a new category of bugs where we fail to correctly express
whether work is intentional or not!
In particular, failing to correctly mark the end of intentional work
seems like a real danger.

### Assigning initiators

RUM providers often want to wrap event handlers in their own code.
This means that if we are not careful,
we could report all events as being RUM code.

## Potential solution

This describes an extension of the `PerformanceObserver` API
that provides information on how much work was done
over set periods of time,
breaking that work to various categories.

### Data

```web-idl
dictionary Work {
  // The initiator of the work done.
  // See below for details.
  DOMString name;
  // The number buckets in this interval in which work was done by this initiator.
  // 0 <= work <= 1.
  unsigned long workedBuckets;
  // The number of distinct work events in this interval by this initiator.
    unsigned long workCount;
  // The amount of ms of CPU time used in this interval by this initiator.
  DOMHighResTimeStamp: cpuMs;
  // The duration of longest contiguous span of time where no work was done by this initiator.
  DOMHighResTimeStamp longestIdleMs;

  // A breakdown of the work into further sub-categories of initiator.
  FrozenArray<Work> children;
}

// Data from this API aggregates over an interval of time.
dictionary Interval {
  // The duration of the interval used for aggregation.
  DOMHighResTimeStamp durationMs;
  // The start time for aggregation of this interval.
  DOMHighResTimeStamp startTimestamp;
  // How many buckets there are in this interval.
  // The interval is split into this many equal-length buckets of time.
  unsigned long buckets;
  // A breakdown of the work done during this interval into categories.
  Work work;
  // Other metadata about the interval
}

interface PerformanceWork extends PerformanceEntry {
  Interval workInterval;
}
```

#### Initiators

Initiators allow devs to understand what caused the work.
They form a tree, with more information being added at each level.

Examples:
- "all": the top-level category that includes all work.
  - "js": javascript tasks
    - "toplevel": JS which ran from the top-level
    - "setTimeout":
    - "setInterval":
    - "promise":
    - "event": JS which ran from an event
      - event-type: the `.type` of the event
  - "animation": directly attributable to animation
    - name: the `animation-name` property from CSS
  - "style": due to recalculation of CSS styles
  - "layout":
  - "paint":
  - "media": due to media playing

All "js" initiators break down further with
- filename: the file in which the JS lives
  - line and column: the location of the JS in that file

Other initiators could have further breakdown to identify them more specifically.
For example, having animations identify
which element was being animated
would be more helpful
than just the animation name.

### Invocation

Because the work observer allows several specialized options,
we add a new `WorkOptions` interface

```js
dictionary WorkOptions {
  DOMHighResTimeStamp bucketDuration;
  // The requested duration of the interval.
  // Reported intervals may have a different duration to that requested.
  // Intervals may be truncated because they are reported just before `pagehide`.
  // Intervals may be extended because they represent a long period with no work.
  DOMHighResTimeStamp intervalDuration;
  // If true, an entry will be reported immediately every `intervalDuration`,
  // whether or not any work occurred in that interval.
  // The callback to handle the report will be excluded from the report
  // but it will be included as work by any other `PerformanceObserver`.
  // If false, the observer callback will not be called
  // until there is at least one entry that contains work.
  //
  // ***
  // Use with extreme care.
  // This will result in recurring JS callbacks until `disconnect` is called.
  // ***
  boolean reportIdle;
}
```

```js
function workObserver(list, observer, options) {
  for (const entry of entries.getEntries()) {
    console.log(entry.toJSON());
  }
}

const observer = new PerformanceObserver(workObserver);
observer.observe({
  type: "work",
  workOptions: {
    bucketDuration: 10,  // ms
    intervalDuration: 10000,  // ms
  },
});
```

## More info

For now this repo and explainer is a place-holder.
An API is described in this [slide deck](https://docs.google.com/presentation/d/1d8VaGHF9OFF9Kuy--jIUvYLcGouPtJg7Yxtryk3HvMc/edit).
This API was discussed at the [WebPerfWG meeting on 2026-09-10](https://docs.google.com/document/d/10dz_7QM5XCNsGeI63R864lF9gFqlqQD37B4q8Q46LMM/edit?tab=t.0#heading=h.pndss1ey0460)
and this repo has been created to facilitate discussion.
The content from that slide deck will be moved into this explainer.

There are many issues with the API shape of this proposal
- maybe it should align with the JS Profiling API

Right now, the API shape is secondary to figuring out
- what would actually be useful (and used in reality)
- what should be part of the API and what should be left to be implemented in JS around the API

# This explainer is incomplete

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
