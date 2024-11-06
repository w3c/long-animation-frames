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

## Example of the problem

[Try It](https://loaf-wrapper-script.glitch.me/) in a browser that supports Long Animation Frames API.  Click the two buttons, open console, and compare the LoAF entry's `scripts` values.

```html
<button id="original">Original</button>
<button id="wrapper">Wrapper</button>

<script type="module">
  import original_function from "./original_function.js";
  import create_wrapper from "./some_library.js";
  
  original.addEventListener("click", original_function);
  wrapper.addEventListener("click", create_wrapper(original_function));
</script>

<script>
  const observer = new PerformanceObserver(list => {
    for (let entry of list.getEntries()) {
      console.log("LoAF", entry);
    }
  });
  observer.observe({ type: 'long-animation-frame', buffered: true });
</script>
```

Both of these buttons spend the majority of time running `original_function`, but the LoAF attribution looks quite different, depending on which button you click:

Original:
```json
{
    "name": "script",
    "entryType": "script",
    "startTime": 12345,
    "duration": 500,
    "navigationId": "d4ea7b87-5009-4f81-a55b-79ce71c01135",
    "invoker": "BUTTON#original.onclick",
    "invokerType": "event-listener",
    "windowAttribution": "self",
    "executionStart": 12345,
    "forcedStyleAndLayoutDuration": 0,
    "pauseDuration": 0,
    "sourceURL": "https://uneven-ancient-position.glitch.me/original_function.js",
    "sourceFunctionName": "original_function",
    "sourceCharPosition": 143
}
```

Wrapper:
```json
{
    "name": "script",
    "entryType": "script",
    "startTime": 12345,
    "duration": 500,
    "navigationId": "d4ea7b87-5009-4f81-a55b-79ce71c01135",
    "invoker": "BUTTON#wrapper.onclick",
    "invokerType": "event-listener",
    "windowAttribution": "self",
    "executionStart": 12345,
    "forcedStyleAndLayoutDuration": 0,
    "pauseDuration": 0,
    "sourceURL": "https://uneven-ancient-position.glitch.me/some_library.js",
    "sourceFunctionName": "wrapper_function",
    "sourceCharPosition": 85
}
```

Some differences are useful (invoker, etc), but some are not useful (sourceURL, sourceFunctionName, etc).

When reporting from field, you might conclude that `some_library.js` is slow, and miss the fact that it was `original_function`.


## Introducing user-defined entry points

### Declaring a user-defined entry point
To enable more granular insight without incurring the overhead of a sampling profiler, this proposal introduces a way for a caller of a function to declare that that function is an "entry point" for the purpose of performance monitoring.

```js
// Use in a way identical to `Function.prototype.bind`
const bound_function = performance.bind(original_function, ...prepend_arguments)

// Or give a name and additional options
const bound_function = performance.bind({
  callback: original_function,
  name: "myEventListener"
  prependArguments: []
})

// "wrapper" functions remain the default entry points
element.addEventListener("click", function unrelated_wrapper_function(e) {
  // "wrapper" functions often do some work that remains attributed to them
  doSomeWork();

  // Calling original_function() would attribute to wrapper_function()
  // But by calling bound_function(), a new entry point is created
  bound_function(e);
});
```

During the execution of a performance-bound function, the time spent would be counted as part of the execution of that function, as if it was a script entry point.

### How it would appear in the performance timeline
When present, user-defined entry points would appear in the `scripts` array of a `PerformanceLongAnimationFrameTiming` entry, alongside other long scripts (>5ms).  This now means that the same portion of time may be reported to multiple scripts (i.e. creating a very simple "stack").

Because now entries can overlap in time, an attribute would be added to the script entry, `selfDuration`, which would report the duration specifically for that entry, with all nested entry durations subtracted:

```js
new PerformanceObserver(entries => {
  const entry = entries.getEntries()[0];
  const [outer_script, inner_script] = entry.scripts;

  // "event-listener"
  outer_script.invokerType;

  // "outer_function"
  entry.scripts[0].sourceFunctionName;

  // when `outer_function` was called
  entry.scripts[0].startTime;

  // The duration from `startTime` up until the moment the microtask queue was empty.
  entry.scripts[0].duration;

  // The amount of time spent in `outer_function` and its microtasks, not including bound functions.
  entry.scripts[0].selfDuration;

  // "user-bound" (*bikesheddable)
  outer_script.invokerType;

  // "inner_function"
  entry.scripts[1].sourceFunctionName;

  // when `inner_function` was called
  entry.scripts[1].startTime;

` // the duration between `startTime` and the time `inner_function` returned.
  entry.scripts[1].duration;

  // The total duration of `inner_function` including microtasks.
  entry.scripts[1].selfDuration;
}).observe({type: "long-animation-frame"});

```
It would appear in the [`scripts`](https://w3c.github.io/long-animation-frames/#dom-performancelonganimationframetiming-scripts) array of a `long-animation-frame` performance entry.


## Some details

### Overlaps
The performance timeline is not opinionated about having multiple or overlapping user-defined entry points. If multiple performance-bound functions run concurrently, they would all appear in the timeline.

However, the 5ms threshold for including scripts in the timeline still applies.  Only "long" performance-bound functions (>5ms `selfDuration`) would be included.  The `selfDuration` of all wrapper entry points would subtrack any nested entry points, even if they are not over 5ms.

### Microtasks
While executing a performance-bound function, it might queue microtasks. The time spent in those microtasks, as long as they run within the current task, are attributable to the performance-bound functions that were running at the time of queuing them.
Envision the following scenario:

```js
async function original_function() {
  await some_resolved_promise;
  do_an_app_thing();
}

const bound_function = performance.bind(original_function);

async function wrapper_function() => {
  bound_function();

  await some_other_resolved_promise;
  do_a_wrapper_thing();
};
```

In the above example, both the wrapper and the original functions would queue microtasks, which might contribute to main-thread bottlenecks.

With `performance.bind()` it is guaranteed in this case that the time spent in `do_a_wrapper_thing` would be attributed to `wrapper_function`, and the time spent in `do_an_app_thing` would be attributed to both `original_function` and `wrapper_function`.

Note that `duration` has a different meaning for default entry points than for user-defined entry points:
- For default entry points, the duration includes the time including *all* microtasks.
- For user-defined entry points, the duration is the time before the microtask checkpoint.

That's because the time spent in the inner function might not be contiguous.  Generally speaking, `duration` can help us draw the script entry on the timeline, and `selfDuration` matters more in terms of attribution.

## Alternative considered

### User timing

It is already possible to annotate *anything* using [user timing](https://developer.mozilla.org/en-US/docs/Web/API/Performance_API/User_timing).
However, user timing has some major drawbacks:

- It's a writable global buffer. Any script with access to the `window` object can write entries to it. So it's difficult to use it as a source of truth for attribution.
- User timing doesn't have a threshold. The user is responsible for measuring using `performance.now()` and deciding what goes into the buffer. This could lead to very wasteful buffer or processing overhead.
- There is no way (for a wrapper) to measure and attribute time outside of synchronous function call (i.e. time spent in microtasks).

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





