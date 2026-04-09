---
title: "Event Loop in JavaScript Explained — Async Order Without Guessing"
summary: "JavaScript often surprises developers with the way it executes asynchronous code. The reason behind this confusion is the event loop — the mechanism that decides when and how JavaScript runs async code."
publishedAt: 2026-02-25
---

JavaScript often surprises developers with the way it executes asynchronous
code. Sometimes something doesn't work as expected, and the magical fix turns out to
be wrapping a piece of code inside `setTimeout()`. This can be a source of headaches even for experienced developers.

The reason behind this confusion is the **event loop** — the mechanism that decides when and how JavaScript runs asynchronous code.

# JavaScript Is Single-Threaded

Inside the browser and in Node.js, JavaScript runs on a single thread. This
means only one call stack is available at any given time.

What this means in practice: synchronous code blocks everything else. For heavy,
time-consuming operations, JavaScript cannot do anything else until the current
task finishes.

```javascript
console.log("A");

setTimeout(() => console.log("B"), 0);

const start = Date.now();
while (Date.now() - start < 2000) {} // block ~2 seconds

console.log("C");
```

Even though the timeout is set to zero, it cannot execute while the stack is
occupied by synchronous code. That's why the output order is: `A`, `C`, `B`.

# Web APIs: `fetch`, `setTimeout`, and Friends

If JavaScript is single-threaded, how can it do async things like network
requests or timers?

Because the JavaScript engine doesn't do those things alone. In the browser,
there are **Web APIs** provided by the environment: timers, DOM events, networking, storage, and more.
Thanks to them, JavaScript is able to delegate asynchronous tasks to the
browser and avoid blocking the thread.

The key idea:

- JavaScript schedules work with a Web API (e.g., a network request, a timer).
- The Web API does the background work.
- When it's ready, it schedules a callback/continuation to run later on the main JS thread.

# Callback Async Example: Task Queue + Event Loop

Let's start with the classic:

```javascript
console.log("A");

setTimeout(() => console.log("B"), 0);

console.log("C");
```

Step-by-step:

- `"A"` prints immediately (normal synchronous execution).
- `setTimeout` registers the timer in the Web API area; the callback does not run now.
- `"C"` prints.
- After the call stack becomes empty and the timer has expired, the callback is
  placed into the **task queue** (also called the macrotask queue).
- The event loop notices the stack is empty, takes the next task, pushes it onto
  the stack, and runs it — printing `"B"`.

Output: `A`, `C`, `B`.

A good mental picture is three places:

- **Call stack** — what is currently running
- **Web APIs** — where the waiting happens
- **Task queue** — callbacks waiting to be executed

## Promises and the Microtask Queue

Promises introduce another queue: the **microtask queue**.

Microtasks include:

- Promise reactions (`.then`, `.catch`, `.finally`)
- `queueMicrotask(...)`

**Microtasks have higher priority than tasks.**

A simple demonstration:

```javascript
console.log("A");

setTimeout(() => console.log("timeout"), 0);

Promise.resolve().then(() => console.log("promise"));

console.log("B");
```

What happens:

- Synchronous logs run first → `"A"`, `"B"`
- When the stack is empty, the event loop drains microtasks → `"promise"`
- Only when the microtask queue is empty does the event loop run the next task → `"timeout"`

This "microtasks first" rule is the main reason Promise-based code can appear to "jump ahead" of timers.

# Event Loop: How It Works (Microtasks Prioritized)

You can predict most real-world execution order with a small set of rules:

1. Run all synchronous code until the call stack is empty.
2. Drain the microtask queue completely.
3. Take one task from the task queue and run it.
4. Drain microtasks again.
5. Between tasks, the browser may render (repaint) and handle other internal steps.

The important part: microtasks are drained completely before even the first task is handled.

# What Will Be Printed? (Challenge)

Try to predict the output before looking at the explanation.

## Challenge 1

```javascript
console.log(1);

setTimeout(() => console.log(2), 0);

console.log(3);
```

<details>
<summary>Click for answer</summary>
"1", "3", "2"
</details>

## Challenge 2

```javascript
console.log(1);

setTimeout(() => console.log(2), 0);

Promise.resolve().then(() => console.log(3));

console.log(4);
```

<details>
<summary>Click for answer</summary>
"1", "4", "3", "2"
</details>

## Challenge 3

```javascript
setTimeout(() => {
  console.log("T1");
  Promise.resolve().then(() => console.log("M1"));
}, 0);

Promise.resolve().then(() => {
  console.log("M0");
  setTimeout(() => console.log("T2"), 0);
});

console.log("S");
```

<details>
<summary>Click for answer</summary>
<p>"S", "M0", "T1", "M1", "T2"</p>
<p>- "S" is sync.</p>
<p>- Drain microtasks → "M0" runs; it schedules "T2" (a future task).</p>
<p>- Next task → "T1" runs; inside it, a microtask "M1" is scheduled.</p>
<p>- After that task finishes, drain microtasks → "M1".</p>
<p>- Next task → "T2".</p>
</details>

## Challenge 4 (async/await)

```javascript
async function run() {
  console.log("A");
  await null;
  console.log("B");
}

console.log("S");
run();
console.log("E");
```

<details>
<summary>Click for answer</summary>
"S", "A", "E", "B"
<p>Reason: <code>run()</code> starts synchronously until <code>await</code>. <code>await</code> yields and schedules the rest as a microtask.</p>
</details>

# Wrap Up

JavaScript runs your code on a single call stack, so synchronous code blocks
everything. Async behavior comes from the environment (Web APIs / platform) that
schedules callbacks to run later. The event loop decides when those callbacks run,
and the most important rule to remember is: **microtasks (Promises) run before tasks (timers/events)** once the stack is empty.

If you internalize **stack → drain microtasks → run one task → drain microtasks**, you'll be able to reason about most execution order bugs without guessing.
