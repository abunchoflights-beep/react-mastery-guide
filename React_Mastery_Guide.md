# React for interviews — and how to put it into xBeliv

Current as of August 2026: **React 19.2**, **Next.js 16**. Class components are legacy; nobody will ask you to write one. Server Components, Suspense and the React 19 Actions family are what separates candidates now.

---

## 0. The shape of a React interview

You will not be asked to build a todo app. Expect roughly this:

1. **Concept questions** — "what causes a re-render", "why does this effect run twice"
2. **A planted-bug round** — they show you 15 lines with a deliberate mistake and watch how you reason
3. **A judgment question** — "server or client component here, and why?"
4. **Your own project** — they'll open xbeliv.com and ask you to explain a decision

Section 5 is the one that decides the outcome. Most candidates can recite hooks and cannot debug.

---

## 1. Rendering — the questions you will definitely get

### What causes a component to re-render?

Three things: its own state changes, its parent re-renders, or a context it consumes changes value.

The trap: **props changing is not a cause.** When a parent re-renders, its children re-render regardless of whether their props changed. Memoization is what breaks that chain. Say this out loud in an interview and you're ahead of most applicants.

### Why does my effect run twice in development?

React StrictMode intentionally double-invokes effects in dev to surface missing cleanup. It's a test, not a bug — if double-running breaks your effect, your effect is missing a cleanup function. It does not happen in production.

### Keys

```jsx
{messages.map((m, i) => <Message key={i} {...m} />)}   // wrong
{messages.map(m => <Message key={m.id} {...m} />)}     // right
```

Index keys break the moment a list reorders or gets an item inserted at the top. React reconciles by key, so it reuses the wrong DOM node — and any component state (an open menu, a focused input, a half-typed reply) travels to the wrong item. A chat inbox that prepends new messages is the textbook case where this blows up visibly.

### useMemo and useCallback

They do not make anything faster by default. They preserve *referential identity* across renders. They're only useful when:

- the value is passed to a `React.memo`'d child, or
- the value sits in another hook's dependency array, or
- the computation is genuinely expensive

Wrapping everything in `useCallback` is a common junior tell. And the senior-flavoured follow-up: the **React Compiler** (supported in Next.js 16) auto-memoizes, which is making manual memoization largely unnecessary in new code.

---

## 2. Hooks — where the bugs live

### The race condition (learn this one cold)

```jsx
useEffect(() => {
  let ignore = false;
  fetch(`/api/conversations/${id}/messages`)
    .then(r => r.json())
    .then(data => { if (!ignore) setMessages(data); });
  return () => { ignore = true; };
}, [id]);
```   

**Why:** a user clicks conversation A, then B a moment later. A's response can arrive *after* B's and overwrite it — B is on screen, A's messages are in it. The cleanup flag discards the stale response. `AbortController` is the stronger version because it also cancels the request.

This is the single most-asked practical React question, and you have a real-time inbox, so you will be asked it about your own product.

### The stale closure

```jsx
useEffect(() => {
  const t = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(t);
}, []);   // count is frozen at its first-render value: 0
```

Fix with the functional update — `setCount(c => c + 1)` — which reads the latest state instead of the captured one. Empty dependency array plus a value from scope is the shape to recognise.

### Derived state — the most common mistake in real codebases

```jsx
// wrong: state that mirrors other state
const [filtered, setFiltered] = useState([]);
useEffect(() => { setFiltered(items.filter(i => i.unread)); }, [items]);

// right: just compute it
const filtered = items.filter(i => i.unread);
```

Rule to say out loud: **if you can calculate it from existing state or props, it isn't state.** The wrong version causes an extra render and can display stale data for one frame.

### useRef

A mutable box that survives re-renders and does *not* trigger one when it changes. Two uses: DOM access, and storing something (a timer id, a previous value, a websocket) that shouldn't drive rendering.

### Context and re-renders

Every consumer re-renders when the context *value identity* changes. So this re-renders the whole tree on every parent render:

```jsx
<AuthContext.Provider value={{ user, role, status }}>   // new object each render
```

Fixes: `useMemo` the value, or split into separate contexts so a component that only needs `role` doesn't re-render when something else changes. Directly relevant to your role/status gating in xBeliv — expect to be asked about it.

### Rules of hooks, and why

Top level only, never in a condition or loop. The reason — React tracks hooks by *call order*, not by name. Skip one on a later render and every subsequent hook reads the wrong slot.

---

## 3. React 19 — the fluency check

| API | What it does | Where you'd use it |
|---|---|---|
| `use()` | Reads a promise or context *during render*; unlike hooks, it can be called conditionally | Unwrapping a promise passed from a server component |
| Actions | Async functions run in a transition, with pending and error state handled for you | Form submits |
| `useActionState(fn, initial)` | Returns `[state, formAction, isPending]` | Wiring a form to a server action with error display |
| `useFormStatus()` | Reads the parent form's pending state | A submit button that disables itself, without prop drilling |
| `useOptimistic(state, fn)` | Shows an optimistic value until the real result lands | Sending a chat message |
| `ref` as a prop | `forwardRef` is no longer needed | Any component that forwards a ref |
| `<title>`, `<meta>` in components | Hoisted to `<head>` automatically | Page metadata |

`useOptimistic` is the one to demo. Your inbox is the canonical use case: the message appears instantly, and reverts if the send fails.

---

## 4. Server Components — the topic that decides Next.js interviews

**Server Components are the default in the App Router.** `"use client"` marks a boundary: that file and *everything it imports* becomes client code.

| | Server Component | Client Component |
|---|---|---|
| Can be `async` / await data | Yes | No |
| Hooks (`useState`, `useEffect`) | No | Yes |
| Event handlers (`onClick`) | No | Yes |
| Hit the database directly | Yes | No |
| Ships JavaScript to the browser | No | Yes |

**The mistake interviewers look for:** putting `"use client"` at the top of a page because one button needs an `onClick`, which drags the entire subtree into the bundle. The right move is to push the boundary *down* to the smallest leaf that needs interactivity.

**The escape hatch worth knowing:** a client component can render server components passed to it as `children`. The children are rendered on the server and slotted in — so a client-side layout shell can still wrap server-rendered content.

```jsx
// ClientShell is "use client", but ServerContent stays on the server
<ClientShell>
  <ServerContent />
</ClientShell>
```

**Props crossing the boundary must be serializable.** No functions, no class instances. Passing a callback from a server component to a client component throws — a very common planted bug.

### Also expect

- **Suspense and streaming** — `loading.tsx` is a Suspense boundary; wrap slow sections so the shell paints immediately
- **`error.tsx`** — an error boundary per route segment; must be a client component
- **Server Actions vs Route Handlers** — actions for mutations from your own UI, route handlers for webhooks and third-party callers. You have both in xBeliv: your Meta webhook is a route handler by necessity
- **Caching** — Next.js 16's Cache Components and `use cache`, and knowing when a route goes dynamic. This is where candidates get taken apart; be honest about what you've actually configured

---

## 5. The planted-bug round

Train yourself to spot these on sight. Read the code looking for them in this order:

1. `key={index}` on a list that can reorder
2. `useEffect` fetching with no cleanup → race condition
3. Empty dependency array using a value from scope → stale closure
4. State mutated directly (`items.push(x)` then `setItems(items)`) → same reference, no re-render
5. `useState` + `useEffect` computing derived state
6. Missing cleanup on a subscription, interval or listener
7. A hook inside an `if`
8. Context value as an inline object literal
9. `"use client"` at the top of a tree that doesn't need it
10. A function prop passed from a server to a client component

Practise saying it as: *"I'd check X first — here's what would break, and here's the fix."* Reasoning out loud scores higher than a correct silent answer.

---

## 6. Put it in xBeliv — eight concrete tasks

Each one produces a real improvement *and* an interview answer about your own code.

1. **Audit every list for index keys.** Inbox, conversation list, message list. Switch to stable IDs. → keys and reconciliation
2. **Add the `ignore` flag or an AbortController** to every effect that fetches on conversation change. → race conditions
3. **Add `useOptimistic` to message send.** Message renders instantly, rolls back on failure. → React 19 Actions
4. **Split your auth context** into what changes often and what doesn't, and memoize the value. Measure the re-render drop with React DevTools Profiler. → context performance
5. **Walk the `"use client"` boundaries.** Find the highest one and push it down. Note the bundle-size difference before and after — that's a number you can quote in an interview
6. **Wrap the inbox in Suspense with a `loading.tsx`** so the shell streams in before messages resolve. → streaming
7. **Convert one form to a Server Action** with `useActionState` for errors and `useFormStatus` on the submit button. → forms in React 19
8. **Add `error.tsx`** to the inbox route so an LLM provider failure degrades gracefully instead of blanking the page. → error boundaries

Do them in that order. One to two per session. Commit each separately with a clear message — your commit history becomes evidence.

---

## 7. Two-week drill

**Week 1 — fundamentals.** One concept per day from sections 1 and 2, and implement the matching xBeliv task the same day. Reading without building doesn't stick and doesn't survive follow-up questions.

**Week 2 — modern React.** Sections 3 and 4, tasks 5 through 8. Then record yourself for ten minutes explaining your `"use client"` boundary decisions in English. Listen back once. Do it again.

**Daily, 15 minutes:** open a random file in your own repo and find one of the ten bugs from section 5. There will be some. Finding them in your own code is the exact skill the bug round tests.

---

## 8. What to say when you don't know

You will hit a question you can't answer — probably on caching. The answer that keeps you in the room:

> "I haven't configured that directly. In xBeliv I've handled it by [what you actually did]. My understanding is [what you do know] — is that the direction you'd take?"

Interviewers hire people who can say that. The alternative is confident invention, and they can always tell.

---

Sources worth bookmarking: [React 19 release notes](https://react.dev/blog/2024/12/05/react-19) · [Next.js 16](https://nextjs.org/blog/next-16) · [greatfrontend React question list](https://github.com/greatfrontend/top-reactjs-interview-questions)
