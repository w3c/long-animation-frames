# User-defined script entry point

## Overview
When monitoring main-thread performance, script execution is a common source of bottlenecks.
The web platform currently provides two mechanism to monitor scripts:
- [JS Self profiling](https://wicg.github.io/js-self-profiling/)
- [Long animation frame script entry-points](https://w3c.github.io/long-animation-frames/#sec-PerformanceScriptTiming).

While self-profiling provides all the necessary information, it is highly verbose, has overhead, and is sometimes complex to analyze.
On the other hand, long animation frame scripts are "entry-points": they include the entire time span that a callback/event/script-block was run, until the microtask queue is flushed.
While this is useful in many situations, and has acceptable overhead for a field metric, there are some cases in which a script entry point is not granular enough to be useful.
For example, a framework or error reporting library can wrap all the event listeners for a web page, appearing as the "entry point" in the timeline, and the true culprit for the bottleneck would be obfuscated.

## Introducing user-defined entry points
To enable more granular insight without incurring the overhead of a sampling profiler, proposing to introduce a way for a caller of a function to declare that that function is an "entry point" for the purpose of performance monitoring.

```js
// Use in a way identical to `Function.prototype.bind`
const outer_function = performance.bind(inner_function, ...prepend_arguments)

// Or give a name and additional options
const outer_function = performance.bind({
  callback: inner_function,
  name: "myEventListener"
  prependArguments: []
})

// Use the new function in place of the old one
element.addEventListener("click", e => outer_function(e));
```

During the execution of a performance-bound function, the time spent would be counted as part of the execution of that function, as if it was a script entry point.
It would appear in the [`scripts`](https://w3c.github.io/long-animation-frames/#dom-performancelonganimationframetiming-scripts) array of a `long-animation-frame` performance entry.

## Some details
### Overlaps
The performance timeline is not opinionated about having multiple or overlapping user-defined entry points. If multiple performance-bound functions run concurrently, they would all appear in the timeline.
However, the 5ms threshold for including scripts in the timeline still applies - only "long" performance-bound functions that are part of long animation frames would appear in the timeline.

### Microtasks
While executing a performance-bound function, it might queue microtasks. The time spent in those microtasks, as long as they run within the current task, are attributable to the performance-bound functions that were running at the time of queuing them.
Envision the following scenario:

```js
async function app_callback() {
  await Promise.resolve();
  do_an_app_thing();
}
const app_calllback_wrapper = performance.bind(app_callback);
setTimeout(function timeout_wrapper() => {
  Promise.resolve().then(() => { do_a_wrapper_thing(); }
  app_calllback_wrapper();
}, 1000);
```

In the above example, both the wrapper and the app would queue microtasks, which might contribute to main-thread bottlenecks.
With `performance.bind()` it is guaranteed in this case that the time spent in `do_a_wrapper_thing` would be attributed to `timeout_wrapper`, and the time spent in `do_an_app_thing` would be attributed to both `app_callback` and `timeout_wrapper`.

## Alternative considered

### User timing

It is already possible to annotate *anything* using [user timing](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API/User_timing).
However, user timing has some major drawbacks:

- It's a writable global buffer. Any script with access to the `window` object can write entries to it. So it's difficult to use it as a source of truth for attribution.
- User timing doesn't have a threshold. The user is responsible for measuring using `performance.now()` and deciding what goes into the buffer. This could lead to very wasteful buffer or processing overhead.

### Figure out the hot functions automatically

There is no current solution for this that doesn't have overhead similar to JS self profiling.

## Security & privacy self review 

See [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/)

### 01. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

It exposes timing of functions defined by the web site or party, including queued microtasks that run in the same task.

#### 02. Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes.

#### 03. How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

This feature does not deal with personal information.

#### 04. How do the features in your specification deal with sensitive information?

This feature does not deal with sensitive information.

#### 05. Do the features in your specification introduce new state for an origin that persists across browsing sessions?

No. This feature only applies to the current document.

#### 06. Do the features in your specification expose information about the underlying platform to origins?

No, apart from timing as already exposed by the high-resolution monotonic clock.

#### 07. Does this specification allow an origin to send data to the underlying platform?

No.

#### 08. Do features in this specification allow an origin access to sensors on a user’s device?

No.

#### 09. What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.

Timing information only.

#### 10. Do feautres in this specification enable new script execution/loading mechanisms?

No.

#### 11. Do features in this specification allow an origin to access other devices?

No.

#### 12. Do features in this specification allow an origin some measure of control over a user agent's native UI?

None.

#### 13. What temporary identifiers do the features in this specification create or expose to the web?

None.

#### 14. How does this specification distinguish between behavior in first-party and third-party contexts?

N/A

#### 15. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

The feature is unaffected by these modes.

#### 16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

Yes.

#### 17. Do features in your specification enable origins to downgrade default security protections?

No.

#### 18. What should this questionnaire have asked?

The questionnaire asked for sufficient information.





