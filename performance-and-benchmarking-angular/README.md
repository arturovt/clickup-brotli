# Table of contents

- [The problem behind the performance browser API](#the-problem-behind-the-performance-browser-api)
- [Small theory](#small-theory)
  * [Latency](#latency)
  * [Throughput](#throughput)
- [Angular built-in profiler](#angular-built-in-profiler)
- [Going deeper](#going-deeper)
  * [Bypassing OnPush checks in ViewEngine](#bypassing-onpush-checks-in-viewengine)
  * [Bypassing OnPush checks in Ivy](#bypassing-onpush-checks-in-ivy)
- [Tracing the number of change detections](#tracing-the-number-of-change-detections)
  * [ViewEngine](#viewengine)
  * [Ivy](#ivy)
- [Tracing layout updates](#tracing-layout-updates)

## The problem behind the performance browser API

The `performance` (and especially `performance.measure`) API depends on computer speed and has a high variance (dispersion) factor. I noticed this several times that results were different depending on the load on my computer. We could still get the average value for some specific component, which is good 👌.

Such measurements are called _metastable_, in order to achieve a _stable_ measurement, a perfect balance of computer resources is needed.

## Small theory

Basically, the speed of systems can be characterized by such metrics as latency and throughput:

- _latency_ is the time it takes for **one** _global change detection_ to pass through the entire tree from the root component and until the very last
- _throughput_ is the number of _global change detections_ that can be run in fixed time

It's enough to understand that _throughput_ can be increased by decreasing _latency_. In order to reduce the _latency_, we need to reduce the number of _local change detections_ (since the _global change detection_ consists of many _local change detections_ for each component in the tree).

### Latency

![Latency](./docs/latency.png)

In the above example the latency will equal to the sum of timings per each local change detection. There are 8 local change detections:

![Latency formula](./docs/latency-formula.png)

### Throughput

In a system where the code is executed in one thread, the throughput is calculated using the following formula:

![Throughput formula](./docs/throughput-formula.png)

## Angular built-in profiler

Angular already has a built-in change detection profiler which can be enabled in the development mode:

```ts
async function bootstrap() {
  await window.clickupCanBootstrapPromise;

  const { AppModule } = await import(
    /* webpackMode: 'eager' */ './app/app.module'
  );

  const { injector } = await platformBrowserDynamic().bootstrapModule(
    AppModule
  );

  if (isDevMode()) {
    const { enableDebugTools } = await import('@angular/platform-browser');
    const { components } = injector.get(ApplicationRef);
    enableDebugTools(components[0]);
  }
}

bootstrap();
```

> ⚠️ Such code can be shipped to the repository only with `if (ngDevMode)` condition. `isDevMode()` is a runtime function which will not be tree-shaken away by Terser, thus `enableDebugTools` and `AngularProfiler` will be bundled into the production bundle.

Therefore it will be accessible in the `window.ng` property. Let's open the DevTools and run it (note that I'm on the notifications page):

```js
ng.profiler.timeChangeDetection({ record: true });
```

![Angular profiler result](./docs/angular-profiler-result.png)

> ⚠️ The profiler runs ticks during 500 ms.

> ⚠️ The profiler doesn't take into account that there can be `OnPush` components, even if the root component is marked as `OnPush` then the `ApplicationRef.tick()` will act as a noop.

Considering the above image we can calcuate the latency and throughput. Given the latency is 0.04 (ms per `tick()`), then the throughput will equal `1000 (ms in 1 second) / 0.04 = 25000` (change detections per second).

## Going deeper

We can still use the same Angular's built-in profiler but we need to bypass `OnPush` checks.

### Bypassing OnPush checks in ViewEngine

There is a function called `callViewAction` that does `OnPush` checks:

```js
function callViewAction(view, action) {
  switch (action) {
    case ViewAction.CheckAndUpdate:
      if ((viewState & 128) /* Destroyed */ === 0) {
        if (
          (viewState & 12) /* CatDetectChanges */ ===
          12 /* CatDetectChanges */
        ) {
          checkAndUpdateView(view);
        } else if (viewState & 64 /* CheckProjectedViews */) {
          execProjectedViewsAction(
            view,
            ViewAction.CheckAndUpdateProjectedViews
          );
        }
      }
  }
}
```

`CatDetectChanges` equals `Attached | ChecksEnabled`, which is `8 (Attached) | 4 (ChecksEnabled) = 12`. Thus `(view.state & 12) === 12` will equal `true` when the view is attached to the change detection tree (it can be detached by calling `detach()` on the `ChangeDetectorRef`) and _checks are enabled_. When do checks become enabled? Angular calls `checkAndUpdateDirectiveInline` which is responsible for checking `@Input()` properties. If any binding has been changed then Angular calls `updateProp` which changes the view state:

```js
if (view.def.flags & 2 /* OnPush */) {
  view.state |= 8 /* ChecksEnabled */;
}
```

We can swap `if` conditions:

```js
function callViewAction(view, action) {
  switch (action) {
    case ViewAction.CheckAndUpdate:
      if ((viewState & 128) /* Destroyed */ === 0) {
        if (viewState & 64 /* CheckProjectedViews */) {
          execProjectedViewsAction(
            view,
            ViewAction.CheckAndUpdateProjectedViews
          );
        } else {
          checkAndUpdateView(view);
        }
      }
  }
}
```

### Bypassing OnPush checks in Ivy

There is a function called `refreshComponents` that does `OnPush` checks:

```js
function refreshComponent(hostLView, componentHostIdx) {
  ...
  if (componentView[FLAGS] & (16 /* CheckAlways */ | 64 /* Dirty */)) {
    refreshView(tView, componentView, tView.template, componentView[CONTEXT]);
  } else if (componentView[TRANSPLANTED_VIEWS_TO_REFRESH] > 0) {
    refreshContainsDirtyView(componentView);
  }
}
```

We can swap `if` conditions:

```js
function refreshComponent(hostLView, componentHostIdx) {
  ...
  if (componentView[TRANSPLANTED_VIEWS_TO_REFRESH] > 0) {
    refreshContainsDirtyView(componentView);
  } else {
    refreshView(tView, componentView, tView.template, componentView[CONTEXT]);
  }
}
```

Let's run the profiler again:

![Angular profiler result without OnPush](./docs/angular-profiler-result-no-onpush.png)

Oh, we can see that the `msPerTick` now differs when `OnPush` cheks are bypassed therefore all components are checked. So now the throughput equals `1000 / 1.57 = 636` (change detections per second).

## Tracing the number of change detections

We have to be able to know how many change detections run per some component.

### ViewEngine

There is a `checkAndUpdate` function which is called for each component when the change detection is run:

```js
function checkAndUpdateView(view) {
  ...
}
```

```js
function checkAndUpdateView(view) {
  const t0 = performance.now();

  // `checkAndUpdateView` body

  if (view.component.constructor !== Object) {
    const name = view.component.constructor.name;
    const t1 = performance.now();
    console.log(
      `%c${t1 - t0} ms`,
      'font-size: 14px; background: red; color: white;',
      ` checkAndUpdateView() took for ${name}`
    );
  }
}
```

### Ivy

There is a `refreshView` function which is called for each component when the change detection is run:

```js
function refreshView(tView, lView, templateFn, context) {
  ...
}
```

We can place the single `console.log` to see how many times it's run for the specific component. Let's store the debuggable element in the global scope:

```js
function refreshView(tView, lView, templateFn, context) {
  const debuggable =
    lView[HOST] !== null &&
    lView[HOST].tagName.toLowerCase() === window.debuggable;
  if (debuggable) {
    console.log(`refreshView() is called for the ${window.debuggable}`);
  }
}

// Run this in the DevTools
window.debuggable = 'cu-task-editor';
```

We can also measure it's execution:

```js
function refreshView(tView, lView, templateFn, context) {
  const debuggable =
    lView[HOST] !== null &&
    lView[HOST].tagName.toLowerCase() === window.debuggable;
  const t0 = debuggable && performance.now();

  // `refreshView` body

  if (debuggable) {
    const t1 = performance.now();
    console.log(
      `%c${t1 - t0} ms`,
      'font-size: 14px; background: red; color: white;',
      ` refreshView() took for ${window.debuggable}`
    );
  }
}
```

But as I said its execution time is _metastable_ and also it will not take a lot of time. We should focus primarily on the **number of calls** of these functions since they lead to layout updates. Less calls = less layout updates. We shouldn't trace the `refreshView` for the root component, but on the contrary, we need to focus on small components that can lead to significant layout updates.

## Tracing layout updates

I think that DevTools probably support that functionality but I'm more a fan of the `chrome-remote-interface` and `puppeteer` packages for tracing purposes. They can give much more information about layout updates and especially its triggers:

<details><summary>Collapse trace.ts</summary>
<p>

```ts
import fs = require('fs');
import CDP = require('chrome-remote-interface');

const categories = [
  '-*',
  'devtools.timeline',
  'disabled-by-default-devtools.timeline',
  'disabled-by-default-devtools.timeline.stack',
];

const enum LayoutUpdateEvent {
  Layout = 'Layout',
  UpdateLayoutTree = 'UpdateLayoutTree',
}

interface DevToolsEvent {
  cat: string;
  dur: number;
  name: string;
  ph: string;
  pid: number;
  tdur: number;
  tid: number;
  ts: number;
  tts: number;
  args: any;
}

interface TracingData {
  value: DevToolsEvent[];
}

(async () => {
  const tab = await CDP.New();
  const client = await CDP({ tab });
  const { Page, Tracing } = client;
  const events: DevToolsEvent[] = [];

  await Page.enable();
  await Tracing.start({ categories: categories.join(',') });

  Tracing.dataCollected(({ value }: TracingData) => {
    events.push(...value);
  });

  await Page.navigate({ url: 'http://localhost:4200' });

  await Page.loadEventFired();
  await Tracing.end();
  await Tracing.tracingComplete();

  const layoutUpdateEvents = events
    .filter(
      ({ name }) =>
        name === LayoutUpdateEvent.Layout ||
        name === LayoutUpdateEvent.UpdateLayoutTree
    )
    .filter(({ args }) => !!args?.beginData?.stackTrace?.length);

  fs.writeFileSync(
    `./update-layout-events.${Date.now()}.json`,
    JSON.stringify(layoutUpdateEvents, null, 4)
  );

  await CDP.Close({ id: tab.id });
})();
```
</p>
</details>

Let's run it:

```shell
$ yarn ts-node  -O "{ \"module\": \"commonjs\" }" --transpile-only chrome.ts
$ echo "There have been $(cat update-layout-events* | jq length) layout updates"
```

Now let's open this file, we can see that there have been 2 layout updates during the page load (only ClickUp loader was shown but not the whole application):

```json
[
    {
        "args": {
            "beginData": {
                "frame": "36DE239E49B204B7B736EF417DDA9480",
                "stackTrace": [
                    {
                        "columnNumber": 17,
                        "functionName": "",
                        "lineNumber": 122735,
                        "scriptId": "31",
                        "url": "http://localhost:4200/vendor.js"
                    }
                ]
            },
            "elementCount": 0
        },
        "cat": "blink,devtools.timeline",
        "dur": 22,
        "name": "UpdateLayoutTree",
        "ph": "X",
        "pid": 2829964,
        "tdur": 22,
        "tid": 1,
        "ts": 399598446271,
        "tts": 401439
    },
    {
        "args": {
            "beginData": {
                "frame": "36DE239E49B204B7B736EF417DDA9480",
                "stackTrace": [
                    {
                        "columnNumber": 76,
                        "functionName": "getHighContrastMode",
                        "lineNumber": 118155,
                        "scriptId": "31",
                        "url": "http://localhost:4200/vendor.js"
                    }
                ]
            },
            "elementCount": 8
        },
        "cat": "blink,devtools.timeline",
        "dur": 267,
        "name": "UpdateLayoutTree",
        "ph": "X",
        "pid": 2829964,
        "tdur": 267,
        "tid": 1,
        "ts": 399602264923,
        "tts": 4189199
    }
]
```

The `getHighContrastMode` is a function called by `@angular/cdk`. Another layout update was caused by this piece of code `"lineNumber": 122735`, if we open `vendor.js` and jump to `122735` we can see that the `promise-window` gets `clientWidth` and triggers another layout update.

The `puppeteer` has richer functionality but it still uses the `chrome-remote-interface` internally. We can open existing pages with `puppeteer`, search for DOM elements and dispatch events and then do the tracing stuff. It's harder to do with `chrome-remote-interface` since it requires providing `x` and `y` for the specific DOM element.