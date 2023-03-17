# remix-auto-session

## What?

remix-auto-session adds a `session` to the `context` that gets passed to your [https://remix.run](Remix) routes.

It automatically writes the session to the response headers... but **only if anything in the request modified the session**.

## Why?

Remix has support for sessions out of the box, but the APIs are very low-level. Each `loader` or `action` that cares about the session must load it and -- if it modifies the session -- must write it to the response headers.

That means every route needs to have at least four concerns:

1. asynchronous session loading and possible error-handling
2. whatever the route is trying to do
3. remembering whether the session was modified and, if modified, writing the session to the response headers in every place it returns a response

Sessions have a _lot_ of uses: authentication, authorization, flash messaging, remembering recently-viewed items, and more. Each of those use-cases causes your route to have to do sessions-related work that remix-auto-session can take care of for you!

## Example

### Before

```ts
// app/routes/newsletter.tsx
import type { ActionArgs } from '@remix-run/node' // or whatever adapter
import { getSession, commitSession } from '~/sessions'

export function action({ request }: ActionArgs) {
  // 1. load the session
  const session = await getSession(request.headers.get('cookie'))

  // 2: do the work the route is made for
  const email = (await request.formData()).get("email")
  try {
    await subscribe(email);
    session.flash('success', 'Thank you for signing up!')

    return json(
      { error: null, ok: true },
      // 3. we modified the session by adding flash, so we need to write it
      { headers: { 'Set-Cookie': await commitSession(session) } },
    )
  } catch (error) {
    session.flash('error', "Sorry, we're having some trouble signing you up")
    return json(
      { error: error.message, ok: false },
      // 3 (again!). we modified the session by adding flash, so we need to write it
      { headers: { 'Set-Cookie': await commitSession(session) } },
    )
  }
}
```

### After

```ts
// app/routes/newsletter.tsx
import type { ActionArgs } from '@remix-run/node' // or whatever adapter
import type { WithAutoSession } from 'remix-auto-session'

export function action({ context }: WithAutoSession<ActionArgs>) {
  // 1: load the session
  const session = await context.session()

  // 2: do the work the route is made for
  const email = (await request.formData()).get("email")
  try {
    await subscribe(email);
    session.flash('success', 'Thank you for signing up!')
    // 3: there is no 3
    return json({ error: null, ok: true })
  } catch (error) {
    session.flash('error', "Sorry, we're having some trouble signing you up")
    // 3: still no 3
    return json({ error: error.message, ok: false })
  }
}
```

## Configuring

Your sold. Great! How do you teach remix-auto-session how to load your session?

### With Node

Your `server.js` looks something like this:

```ts
app.use(morgan("tiny"))
app.all("*", createRequestHandler({ build: require(BUILD_DIR) }))
```

Add the auto-session middleware to the stack:

```ts
import { expressMiddleware as autoSession } from 'remix-auto-session'
import { getSession, commitSession } from "app/sessions"

app.use(morgan("tiny"))
app.use(autoSession({ getSession, commitSession }))
app.all('*', createRequestHandler({ build: require(BUILD_DIR) }))
```

### With Cloudflare

Your `server.js` looks something like this:

```ts
export function onRequest(context) {
  return handleRequest(context)
}
```

Change that to

```ts
import { autoSession } from 'remix-auto-session'
import { getSession, commitSession } from "app/sessions"

export function onRequest(context) {
  return autoSession({ commitSession, getSession, handleRequest })(context)
}
```

## Is it any good?

Not yet. Please open a discussion with your thoughts on how we can make this
great.

## References

* [Remix Sessions](https://remix.run/docs/en/1.14.3/utils/sessions#using-sessions)
* [Remix discussion 5633](https://github.com/remix-run/remix/discussions/5633): CookieManager, SessionManager
