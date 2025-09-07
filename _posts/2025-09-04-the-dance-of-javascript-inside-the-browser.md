---
layout: post
title: The Dance Of JavaScript In the Browser
date: 2025-09-04 15:09:00
description: Usage of JavaScript in real world applications. 
tags: javascript dom performace eventloop
categories: sample-posts
featured: true
---

[High-level concepts you must understand.](#orgb7e72e9)
- [Useful patterns & examples](#orgdbe27ed)
- [Common pitfalls & gotchas](#org569495d)
- [Testing & determinism](#org1fc2abd)
- [Performance & battery considerations](#orgb59fe3e)



<a id="orgb7e72e9"></a>

# High-level concepts you must understand.

1.  **Event loop phases / task queues**
    -   While the event loop itself isn&rsquo;t directly DOM-related, many tasks scheduled by the event loop can be. For instance:
        -   \*Microtask(like `.then` of a `Promise`) can modify the DOM (e.g., updating elements one a `Promise` resolves).
        -   **Macrotasks** (like `setTimeout`, `setInterval`, or `requestAnimationFrame`) can also interact with the DOM by manipulating elements, triggering events, or performing layout/repaint tasks.
        -   `setTimeout` / `setInterval` callbacks are scheduled on the *macrotask* (task) queue (in browsers the &ldquo;timer&rdquo; phase). Microtasks (Promises `.then`, `queueMicrotask`) run before macrotasks that follow the currently executing code.
        -   Consequence: resloving a promise can run before a `setTimeout(..., 0)` handler.
2.  **Zero-delay is not immediate**
    -   `setTimeout(fn, 0)` schedules `fn` to run after current code and any queued microtasks - it is *yielding* control, not instantaneously.
3.  **setTimeout / setInterval Callbacks**:
    -   These are typically used for scheduling DOM updates, such as animations or polling for state changes, so **they are often DOM-related** when used in client-side JavaScript.
    -   For example, you might use `setTimeout` to delay DOM updates after a certain action, or `setInterval` to repeatedly update an element at a fixed interval.
4.  **Clamping and throttling**
    -   These can impact the **performance of DOM updates**. If your&rsquo;re using high-frequency timers to update the DOM, browsers may throttle them when tabs are inactive or when there are too many nested timers. This is particularly important in **animations or continous DOM updates**.
    -   Browsers clamp nested timers or timers in background/inactive tabs. Historically nested timeouts < 4ms get clamped to 4ms; background tabs may be throttled to ~100ms or more. Node has different rules.
5.  **Accuracy and Drift**
    -   Timers that drift can cause issues with **timing-sensitive DOM updates** (like animations or periodic UI updates). For example, if you rely on `setInterval` to update an animation, any drift can result in visible **jank** or inconsistent behavior in the UI.
6.  **Timers and Async Functions**
    -   If you pass asn **async function** to a `setInterval` or `setTimeout`, it won&rsquo;t behave as expected, since the function will run asynchronously. This could lead to race conditions or unexpected behavior when manipulating the DOM.
        
        For example, if the async function updates the DOM, it could result in multiple concurrent updates that might not be desirable.
7.  **Memory Leaks and Closures**
    -   This is very relevant for DOM manipulation, as closures (functions referencing DOM nodes) can **prevent garbage collection** if not cleared properly. If you don&rsquo;t clear your timers (especially if they are tired to DOM elements), you might create memory leaks that **accumulate over time** as the DOM grows.
8.  **Security / DoS / Throttlingh**
    -   While not directly manipulating the DOM, this is relevant to **DOM performance**. Excessive timers can **throttle DOM updates** or cause a page to become unresponsive ( leading to **poor user experience** ). For instance, long-running intervals or timeouts that update the DOM can overwhelm the browser&rsquo;s rendering enginer,, causing frames to drop or UI freezes.


<a id="orgdbe27ed"></a>

# Useful patterns & examples

1.  Promise sleep / `await` friendly
    
    ```js
            function sleep(ms, {signal} = {}){
                return new Promise((resolve, reject) => {
                    if(signal?.aborted) {
                        return reject(new DOMException('Aborted', 'AbortError'));
                    }
                    const id = setTimeout(() => {
                        signal?.removeEventListener('abort', onAbort);
                        resolve();
                    }, ms);
                    function onAbort() {
                        clearTimeout(id);
                        reject(new DOMExecption('Aborted', 'AbortError'));
                    }
                    signal?.addEventListenter('abort', onAbort, {once: true});
                });
            }
    ```
    
    This is better than ad-hoc setTimeouts when you use `async/await` and want cancellation support.

2.  Accurate interval (drift-compensated)
    
    ```js
    function startAccurateInterval(fn, interval) {
        let expected = performance.now() + interval;
        let stopped = false;
    
        function tick() {
            if (stopped) {
                return;
            }
            fn();
            expected += interval;
            const next = Math.max(0, interval - drift);
            setTimeout(tick, next);
        }
        setTimeout(teck, interval);
    
        return () => {
            stopped = true;
        };
    }
    //usage
    const stop = startAccurateInterval(() => console.log('tick', performance.now()), 1000);
    // stop() to cancel
    ```
    
    This corrects for drift caused by callback execution time.

3.  Avoid overlapping `async` interval run
    
    ```js
       let running = false;
       const id = setInterval(async () => {
           if (running) return;
           running = true;
           try {
               await doSomethingAsync();
           } finally {
               running = false;
           }
       }, 1000)
    ```
    
    Or use recursive `setTimeout` to enforce sequential runs.

4.  Exponential backoff + jitter (for retries)
    
    ```js
       function backoff(attemp, base = 200, cap = 10000){
           const exp = Math.min(cp, base * (2 ** attemp));
           // add jitter
           return exp / 2 + Math.random() * (exp / 2)
       }
    ```
    
    Use `setTimeout` with the returned ms. Jittering avoids thundering herd.

5.  Cancelable timers with AbortController
    
    ```js
    function setTimeoutWithSignal(cb, ms, signal) {
        const id = setTimeout(() => {
            signal?.removeEvenListener('abort', onAbort);
            cb();
        },ms)
        function onAbort(){
            clearTimeout(id);
        }
        signal?.addEventListener('abort', onAbort, {once: true});
        return id;
    }
    
    const ac = new AbortController() {
        setTimeoutWithSignal(() => console.log('done'), 5000, ac.signal);
        ac.abort(); // cancels
    }
    ```

6.  Use `requestAnimationFrame` for animations.
    -   `requestAnimationFrame` is synchronized with display refresh rate and pauses in background tabs - use it instead of `setInterval` for visual updates.
7.  Yiedling to the event loop (cooperative blocking) If you must do heavy sychronous work but want to keep UI responsive, break it into slices:
    
    ```js
    async function bigWork(items) {
        for (let i = 0; i<items.length; i++) {
            process(items[i]);
            if(i % 100 === 0){
                await new Promise(r => setTimeout(r, 0)); // yield
            }
        }
    }
    ```


<a id="org569495d"></a>

# Common pitfalls & gotchas

-   **Relying on exact timing** - browsers and OSes will vary. Never assume millisecond-perfect scheduling for non-real-time tasks.
-   **setInterval + long task overlap** - causes multiple simultaneous executions; use locking or recursive scheduling.
-   **Timers keep objects alive** - forgettiing to clear timers tied to DOM nodes leads to leaks.
-   **Using `setInterval` for animations** - leads to frame-skip/jitter; prefer `requestAnimationFrame`.
-   **Nested setTimeout clamping** - repeatedly calling ~setTimeout(&#x2026;, 0) inside handlers can get clamped to a minimum delay.
-   **In tests** - use fake timers(Jest, Sinon) for deterministic behavior. Be aware fake timers change how `Date.now()` and `performance.now()` behave in some libs.


<a id="org1fc2abd"></a>

# Testing & determinism

-   Use Sinon or Jest fake timers to:
    -   Advance time deterministically.
    -   Test backoff and retry logic.
    -   Avoid flakiness in asynchronours tests.
-   But beware: some APIs (like `requestAnimationFrame`, or high-res `performace.now()`) need additional shims or cannot be faked the same way.


<a id="orgb59fe3e"></a>

# Performance & battery considerations

-   Avoid frequent timers on background tabs; check `document.visibilityState` and pause timers when hidden.
-   For background processing use Web Workers or Service Workers where appropriate (and the browser allows longer lifecycle).


