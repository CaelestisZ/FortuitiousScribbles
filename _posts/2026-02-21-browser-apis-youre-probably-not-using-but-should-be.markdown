---
layout: post
title:  "Browser APIs You're Probably Not Using (But Should Be)"
date:   2026-02-21 10:00:00
---

Everyone knows `fetch()`, `localStorage`, and `setTimeout`. They're the bread and butter of web development. But the browser platform has quietly accumulated a treasure trove of powerful APIs that most developers have never touched. Not because they're obscure, but because they rarely make it into tutorials.

This post covers **seven underused browser APIs** that will make you a better web developer. Some handle cross-tab communication. Some let you observe the DOM without polling. Some let you defer work intelligently. All of them are production-ready and supported in modern browsers.

---

## 1. BroadcastChannel

**Cross-tab communication without a server.**

Ever needed two browser tabs to talk to each other? Maybe sync a logout event across all open tabs, or coordinate a shopping cart update? The go-to solution used to be `localStorage` events, which are hacky and limited. **BroadcastChannel is the clean solution.**

It creates a named channel that any tab, window, iframe, or worker on the same origin can subscribe to. Send a message on one end, every subscriber fires. It's pub/sub for browser contexts.

**Great for:** Syncing logout, sharing auth tokens, coordinating cart updates, real-time theme sync.

```javascript
// In every tab, subscribe to the channel
const channel = new BroadcastChannel('app-events');

channel.addEventListener('message', (event) => {
  if (event.data.type === 'LOGOUT') {
    // Every tab hears this, redirect them all
    window.location.href = '/login';
  }
});

// In the tab where the user clicked "Log out"
channel.postMessage({ type: 'LOGOUT' });
channel.close(); // Clean up when done
```

> 💡 **Pro tip:** BroadcastChannel works in Service Workers too, making it excellent for coordinating background sync, push notification handling, or cache invalidation across your entire app.

> ✅ **Browser support:** Chrome, Firefox, Safari, Edge. All modern browsers.

---

## 2. IntersectionObserver

**Know when elements enter the viewport, without the performance hit.**

The old way to detect when an element scrolls into view was attaching a `scroll` event listener and calling `getBoundingClientRect()` on every frame. This is **catastrophically bad for performance**: it forces layout recalculations on the main thread, every single scroll event.

IntersectionObserver does this off the main thread, asynchronously, at the browser level. It's how native lazy loading works. Use it for lazy-loading images, triggering animations, infinite scroll, or analytics tracking (did the user actually see this element?).

```javascript
const observer = new IntersectionObserver((entries) => {
  entries.forEach((entry) => {
    if (entry.isIntersecting) {
      // Element just entered the viewport
      entry.target.classList.add('visible');

      // Stop watching once it's appeared
      observer.unobserve(entry.target);
    }
  });
}, {
  threshold: 0.2,              // Fire when 20% is visible
  rootMargin: '0px 0px -50px 0px' // Shrink the detection zone
});

document.querySelectorAll('.animate-on-scroll')
  .forEach(el => observer.observe(el));
```

> 💡 **Pro tip:** The `threshold` option accepts an array like `[0, 0.5, 1.0]` to fire the callback at multiple visibility milestones. Perfect for tracking how much of an ad or article a user actually read.

> ✅ **Browser support:** Chrome, Firefox, Safari, Edge. All modern browsers.

---

## 3. ResizeObserver

**Element-level resize detection, not just the window.**

The `window resize` event only tells you the window resized. But in today's component-driven UIs, what you really need to know is: **did this specific element change size?** Maybe a sidebar collapsed, content was dynamically injected, or a CSS transition completed.

ResizeObserver watches individual elements and fires with their new dimensions. This enables true container queries in JavaScript, responsive charts, adaptive components, and more. No polling, no resize event hacks.

```javascript
const resizeObserver = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const { width, height } = entry.contentRect;

    // Adapt the component based on its own size
    if (width < 400) {
      entry.target.setAttribute('data-layout', 'compact');
    } else {
      entry.target.setAttribute('data-layout', 'full');
    }
  }
});

resizeObserver.observe(document.querySelector('.my-chart'));
```

> 💡 **Use it for charts:** Libraries like Chart.js and D3 need explicit dimensions. Wrapping them in a ResizeObserver is the cleanest way to make charts genuinely responsive without listening to window resize events.

> ✅ **Browser support:** Chrome, Firefox, Safari, Edge. All modern browsers.

---

## 4. requestIdleCallback

**Schedule non-urgent work for browser idle time.**

The browser has a main thread, and it's precious. Animations, user input, layout: they all compete for it. When you run analytics, prefetching, or non-critical initialization on the main thread during page load, you're stealing time from things users actually feel.

`requestIdleCallback` schedules work to run when the browser is genuinely idle between frames. You even get a deadline object telling you how much time you have, so you can break up long tasks into safe chunks.

```javascript
function doNonCriticalWork(deadline) {
  // Keep working while we have time left
  while (deadline.timeRemaining() > 0 && tasks.length > 0) {
    processTask(tasks.shift());
  }

  // If tasks remain, schedule more idle time
  if (tasks.length > 0) {
    requestIdleCallback(doNonCriticalWork);
  }
}

// Examples of what to defer here:
// - Sending analytics events
// - Pre-rendering off-screen routes
// - Prefetching images for the next page
requestIdleCallback(doNonCriticalWork, { timeout: 2000 });
```

> ⚠️ **Browser support:** Not supported in Safari. Use a lightweight polyfill that falls back to `setTimeout(fn, 0)` in production. The `timeout` option ensures your work eventually runs even on busy pages.

---

## 5. MessageChannel

**Private two-way communication pipelines.**

BroadcastChannel broadcasts to everyone on a channel. But sometimes you want a **private, bidirectional pipe** between exactly two parties. Say, the main thread and a Web Worker, or a parent frame and an embedded iframe.

MessageChannel creates two linked ports. You keep one, send the other to whoever you want to communicate with. Messages flow both ways, privately. It's the backbone of many Worker communication libraries.

> ✅ **Browser support:** Chrome, Firefox, Safari, Edge. All modern browsers.

```javascript
// Main thread, create the channel
const channel = new MessageChannel();

// Transfer port2 to the worker
worker.postMessage({ type: 'INIT' }, [channel.port2]);

// Communicate over port1 (main thread side)
channel.port1.onmessage = (event) => {
  console.log('Worker says:', event.data);
};

channel.port1.postMessage({ cmd: 'process', payload: largeData });

// ---- In the worker ----
self.addEventListener('message', (event) => {
  const port = event.ports[0];
  port.postMessage({ status: 'done', result: computedResult });
});
```

---

## 6. Web Locks API

**Mutex primitives for the browser.**

Concurrency bugs in web apps are sneaky. Multiple tabs making the same API call simultaneously. A service worker and a tab both trying to write to IndexedDB at once. Race conditions that only happen under load. **Web Locks gives you real mutex semantics in the browser.**

Request a named lock, run your critical section, release it. Other code waiting on the same lock queues up. No polling, no flags in localStorage, no custom semaphore hacks.

```javascript
// Only ONE tab can hold "db-write-lock" at a time
await navigator.locks.request('db-write-lock', async (lock) => {
  // This block runs exclusively. No other tab/worker
  // can enter until this async function resolves
  await writeToIndexedDB(data);
  await syncToServer(data);
});
// Lock automatically released when the callback resolves

// Check what locks are currently held:
const state = await navigator.locks.query();
console.log(state.held); // Array of active locks
```

> 💡 **Real-world pattern:** Designate one tab as the "leader" for expensive background tasks (polling, sync, websocket) using a lock. When that tab closes, another tab picks it up automatically. No coordination server needed.

> ✅ **Browser support:** Chrome, Firefox, Safari, Edge. All modern browsers.

---

## 7. Network Information API

**Adapt your app to real network conditions.**

You know your user is on mobile. But are they on 5G in a city or 2G on a mountain? `navigator.connection` exposes the effective connection type, estimated downlink speed, round-trip time, and a **data saver preference flag**.

Use this to serve lower-quality images on slow connections, disable autoplay video, skip prefetching, or switch to a lite mode. All automatically, without asking the user.

```javascript
const conn = navigator.connection;

function adaptToNetwork() {
  const { effectiveType, saveData, downlink } = conn;

  if (saveData || effectiveType === '2g') {
    // User has data saver on or very slow connection
    loadLiteVersion();
  } else if (effectiveType === '4g' && downlink > 5) {
    // Fast connection — prefetch next pages
    prefetchNextRoutes();
  }
}

adaptToNetwork();

// React when network conditions change
conn.addEventListener('change', adaptToNetwork);
```

> 💡 **Check `saveData` first:** The `saveData` boolean is a direct expression of user intent. They've turned on "Data Saver" in their browser or OS settings. Always respect it. Skip analytics, skip autoplay, skip prefetching when it's true.

> ⚠️ **Browser support:** Chrome and Edge only. Not supported in Firefox or Safari. Always guard usage with `if ('connection' in navigator)` and treat it as a progressive enhancement, not a core dependency.

---

## Wrapping Up

The browser platform is richer than you think.

MDN has thousands of documented browser APIs. Most tutorials cover maybe 50 of them. The rest sit quietly in the platform, waiting to be discovered. They often replace entire libraries or solve problems developers currently hack around with ugly workarounds.

The next time you reach for a package to solve a cross-tab sync, scroll detection, or resize problem, check if the browser already has it. It probably does.

