# Explainer for the Unwanted Work API

This proposal is an early design sketch by fergal@chromium.org to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.

## Status

This explainer is a draft proposal seeking feedback.
Feedback on the use cases is currently more desirable
than feedback on the specifics of the API.

Several parts of this explainer are still incomplete.

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

### Capabilities

Allow sites to
- quantify time spent doing work vs idle.
- identify the root causes of unintended work.
- distinguish work by common categories, JS, animation, media, etc.
- distinguish intended work from unintended work.
   - it's not a bug for a movie player to being constantly playing a movie.
   - it's not a bug for a spinner to spin.
     *while* the page is visible and waiting for something to complete.
- distinguish between repeated wake-ups and solid CPU usage.

### Deployability

This API should be low-enough overhead to be always-on
so that unintended work can be discovered and debugged
- early in development and internal dogfood usage
- in the wild

It's not realistic for every site to write code to use this API.
Rather, it should be trivial for sites to include a library
to gather stats via this API
or for RUM providers to provide it as a service.

### End result

This API is intended to be a tool
for sites to achieve long periods of idleness
when the user is not actively engaged with them.

## Non-goals

This is not for
- measuring battery usage/level
- code profiling

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

### Automated testing of idleness

It should be possible to write automated tests
that verify that after performing some interaction,
a page reaches an idle state within some duration.

### Detection of idle state

Pages may have work that they want to do
but do not want that work to interfere with
ongoing user-facing activity in the page.
This API could be used to determine
when the page has reached an idle state.

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

### Not generating more work

It's essential that this API can be used in a way
that allows monitoring of work
without interrupting what would otherwise be
periods of idleness for the page.

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

### How this solution would solve the use cases

### Longitudinal monitoring

The following code would report back (to a server)
the avg number of 10ms buckets that contained work and
the average cpu usage over the lifetime of the page.

This is a pretty crude measurement.
Sites might want to record separate measurements
for different types of work being done.
Some sites might expect to high have media playback
but should still be on the lookout for non-stop JS.

Sites could also easily produce measurements
broken down by whether the page is visible and/or has focus
since the expected amount of work done
is probably quite different depending on those states.
A page that does not reach an idle state while not visible
probably has a bug.
This kind of monitoring is especially important for pages
that are excluded from freezing by browsers
(e.g. pages with chat clients or ongoing media).

```js
function workObserver(list, observer, options) {
  for (const entry of list.getEntries()) {
    const interval = entry.workInterval;
    totalBuckets += interval.buckets;
    totalWorkingBuckets += interval.work.workedBuckets;
    totalTimeMs += interval.duration;
    totalCpuMs += interval.work.cpuMs;
  }
}

function zeroWorkStats() {
  return {
    totalBuckets: 0,
    totalWorkingBuckets: 0,
    totalTimeMs: 0,
    totalCpuMs: 0,
  };
}
var workStats = zeroWorkStats();

const observer = new PerformanceObserver(workObserver);

window.addEventListener("pagehide", () => {
  // Collect any remaining records.
  const list = observer.takeRecords();
  workObserver(list, observer);
  reportWorkStats(
    workStats.totalWorkingBuckets / workStats.totalBuckets,
    workStats.totalCpuMs / workStats.totalTimeMs,
  );
  // The page might go into BFCache and return again.
  // No entries should be generated while in BFCache.
  workStats = zeroWorkStats();
});

observer.observe({
  type: "work",
  bucketDuration: 10,  // ms
  intervalDuration: 10000,  // ms
});
```

### Immediate feedback to devs

The goal here is to identify unwanted work in real-time
and surface this immediately.
This could be through the UI
(if the user is a developer or dev user)
or to cause more detailed logs to be sent to the server
to debug the cause of the extra work.

The `ActivityTracker` class below is not part of the API proposal.
Android provides a [PerformanceMetricsState][performance-metrics-state] library
which allows putting and removing named states.
These states are then included as annotations
on reports from the [JankStats][jank-stats] library
and possibly other performance related libraries.
`ActivityTracker` is something similar.
Something like it *could* be a useful WP API in the future
but for this explainer we just assume a JS class
with the following interface.

```js
interface ActivityTracker {
  // `onExpectationChanged` is called whenever the amount of ongoing activity changes.
  constructor(onExpectationChanged);
  // Returns a dictionary. The keys activity name `strings`
  // and the values are a `number` for the current count for that activity.
  // Activities with a count of `0` are omitted.
  getActivities():
  // Increments the count for `activityName`.
  putActivity(activityName: string);
  // Decrements the count for `activityName`.
  // It is an error to decrement an activity with a `0` count.
  removeActivity(activityName: string);
  // Returns whether there are any activities with a non-zero count.
  hasActivity(): bool;
}
```

```js
let observer;
// Timestamp of when we started observing.
let observerStart;

function findWork(interval, name) {
  for (const child in interval.children) {
    if (interval.name == name) {
      return interval;
    }
  }
}

function idleObserver(list, observer, options) {
  if (!activityTracker.hasActivity()) {
    // While we were waiting, we now expect activity.
    // So we shouldn't expect idleness.
    return;
  }
  // If no work occurs, the observer is not called.
  // So if we are not expecting activity but the observer is called,
  // we should see if it's worth alerting.
  const entries = list.getEntries()
  if (firstInterval) {
    const interval = entries.shift();
    firstInterval = false;
  }
  for (const entry of entries) {
    // Maybe we expect some residual activity in the first 10s.
    if (entry.startTime < observerStart + 10 * 1000) {
      continue;
    }
    const work = findWork(entry.workInterval, "js");
    if (work) {
      alertTheUser(work);
    }
  }
}

function onActivityChanged(activityTracker) {
  if (!activityTracker.hasActivity()) {
    // We have transitioned to expecting no activity.
    // Start monitoring.
    observer = new PerformanceObserver(idleObserver);
    observerStart = Date.now();
    observer.observe({
      type: "work",
      bucketDuration: 10,  // ms
      intervalDuration: 10000,  // ms
    });
  } else {
    if (observer) {
      observer.disconnect();
      observer = null;
    }
  }
}

var activityTracker = new ActivityTracker(onActivityChanged);
```

We could be much more selective
and use the more detailed initiator information
to only alert about work coming from code in our origin etc.

This still leaves us with the problem
of correctly registering and unregistering all work
with the `ActivityTracker`
but that is a problem that is amenable to tooling and automated testing.

### Automated testing of idleness

```js
// Tests whether we reach idleness before `timeoutMs`.
// "idleness" means that we have a period `idleDurationMs` where no work is done.
// This can takes `timeoutMs + idleDurationMs` to test.
function expectIdle(idleDurationMs, timeoutMs) {
  setTimeout(() => detectIdleness(idleDurationMs), timeoutMs);
}

function detectIdleness(idleDurationMs) {
  const observer = new PerformanceObserver(idleObserver);
  observer.observe({
    type: "work",
    bucketDuration: 100,  // ms
    intervalDuration: idleDurationMs + 1000,  // ms
    // Even if no work is done, report the interval proactively.
    reportIdle: true,
  });
};

function idleObserver(list, observer, options) {
  observer.disconnect();
  const entry = list.getEntries()[0];
  if (entry.interval.work.workedBuckets == 0) {
    TestPass();
  } else {
    TestFail();
  }
}
```

### Detection of idle state

To detect when we have reach idleness,
we can
```js
function findWork(interval, name) {
  for (const child in interval.children) {
    if (interval.name == name) {
      return interval;
    }
  }
}

// Calls `onIdle` when (at least) `idleDurationMs` of time passes without JS activity.
function expectIdle(idleDurationMs, onIdle) {
  const idleObserver = (list, observer, options) => {
    for (const entry of list.getEntries()) {
      const work = findWork(entry.workInterval, "js");
      if (!work) {
        observer.disconnect();
        onIdle();
        return;
      }
    }
  }

  const observer = new PerformanceObserver(idleObserver);
  observer.observe({
    type: "work",
    bucketDuration: 100,  // ms
    intervalDuration: idleDurationMs + 1000,  // ms
    // Even if no work is done, report the interval proactively.
    reportIdle: true,
  });
};
```

### Detect expensive or unintended animations

TBD

## Detailed design discussion

Once installed, a `"work"` `PerformanceObserver`
produces `PerformanceWork` records.

### Avoiding work and wake-ups

To minimize the amount of work and wake-ups
caused by using this API,
we do several things
that might be out of line with other `PerformanceObserver`s.

#### Merge idle records

To avoid a situation where a long period of idleness results in a huge number of empty records,
we merge records that are purely idle.
A record that has some work in it should still represent the requested `intervalDuration` and not be extended indefinitely by idle time
but we should avoid presenting consecutive, purely-idle records.

#### Deliver records only when work occurs

If the observer has requested a 10s `intervalDuration`
and work stops after 7s
then we should not proactively deliver the record at the 10s mark.
Instead we should wait until some other work occurs
and post a task to invoke callback with the record.
This way we never wake up the CPU just to process `PerformanceWork` records.

#### Hide the callback's work when `reportIdle` is `true`

If `reportIdle` is `true` then we will proactively call the observer callback
whenever a record is available.
We will not merge idle records.
In that case,
if we were to include the work done running the callback
in the reported work,
it would be harder (but not impossible)
to identify a record that has no work in it.
The callback should still appear as work to other work observers
since it is doing work and waking up the CPU.

This is an ergonomic convenience
and might be dropped if implementation becomes complicated.

<!--

## Considered alternatives

[This should include as many alternatives as you can,
from high level architectural decisions down to alternative naming choices.]

### [Alternative 1]

[Describe an alternative which was considered,
and why you decided against it.]

### [Alternative 2]

[etc.]
 -->
## Security and Privacy Considerations

### Cross-origin scripts

We need to be careful about reporting work done by cross-origin scripts
as this would leak information about their activity
that could not otherwise be found.

If the work is done purely by the cross-origin script
then it should be invisible
or at least anonymized.

If a cross-origin script calls into this origin's code
then this work was detectable by the page
and should be included.
There is a question of what to record.
Should we record only the work done by this origin?

TODO: Investigate how the self-profiler handles this.

TODO: Express this correctly in terms of CORS etc.
Ensure there is a way for scripts to declare that they should be fully transparent.
E.g. work initiated by React.js should be reported
regardless of whether the React code is hosted same-origin or elsewhere.

### Self-review Questionnaire

TODO: Describe any interesting answers you give to the [Security and Privacy Self-Review
Questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/) and any interesting ways that
your feature interacts with [Chromium's Web Platform Security
Guidelines](https://chromium.googlesource.com/chromium/src/+/master/docs/security/web-platform-security-guidelines.md).

## Stakeholder Feedback / Opposition

TBD
<!--
 [Implementors and other stakeholders may already have publicly stated positions on this work. If you can, list them here with links to evidence as appropriate.]

- [Implementor A] : Positive
- [Stakeholder B] : No signals
- [Implementor C] : Negative

[If appropriate, explain the reasons given by other implementors for their concerns.]
-->
# Open issues
There are many issues with the API shape of this proposal
- maybe it should align with the JS Profiling API

Right now, the API shape is secondary to figuring out
- what would actually be useful (and used in reality)
- what should be part of the API and what should be left to be implemented in JS around the API

## References & acknowledgements

TBD

<!-- [Your design will change and be informed by many people; acknowledge them in an ongoing way! It helps build community and, as we only get by through the contributions of many, is only fair.]

[Unless you have a specific reason not to, these should be in alphabetical order.]

Many thanks for valuable feedback and advice from:

- [Person 1]
- [Person 2]
- [etc.]
-->
# History

This API was first proposed this [slide deck][slide-deck].
This was discussed at the [WebPerfWG meeting on 2026-09-10][wg-meeting-2026-09-10]
and this repo has been created to facilitate discussion.
The content from that slide deck is being moved into this explainer.

[performance-metrics-state]: https://developer.android.com/reference/androidx/metrics/performance/PerformanceMetricsState
[jank-stats]: https://developer.android.com/topic/performance/jankstats
[slide-deck]: https://docs.google.com/presentation/d/1d8VaGHF9OFF9Kuy--jIUvYLcGouPtJg7Yxtryk3HvMc/edit
[wg-meeting-2026-09-10]: https://docs.google.com/document/d/10dz_7QM5XCNsGeI63R864lF9gFqlqQD37B4q8Q46LMM/edit?tab=t.0#heading=h.pndss1ey0460
