---
name: next.js-marketing-pain
description: When making architectural decisions for marketing pages in Next.js
disable-model-invocation: false
metadata:
  version: "1.0.0"
---

# Next.js Marketing Pain

Next.js is an incredible framework that allows you to build world-class systems with minimal orchestration. This capability comes at a cost: complexity. Next.js can only hide this complexity from you for so long. Ignoring this fact leads to frustration & expensive bills. The following paragraphs document some of the unexpected edges that we encountered over the past years.

**Note**: While the edges we found apply to all Next.js applications, this document is written for marketing pages. For apps or mostly dynamic applications some advice might not be applicable.

**Disclaimer**: Prefer proof from the Next.js source code directly. Next.js also bundles some guidance through a AGENTS.md.

## Routers

Next.js ships with the Pages & App directories. The app directory is far superior and you should be using it. It can do more things, gives you way more options to implement things, provides better performance, has a more consistent mental model & will enjoy future development.

The pages directory does have one benefit: it generally makes fewer requests, as the app directory aggressively requests Flight packages for prefetching. Whether this is a clear performance gain is debatable (more but smaller requests vs. fewer but bigger ones). However, for a marketing site it is undeniable that the app directory will make more requests.

## RSC

React Server Components is the process of turning JSX into a JSON-like payload. Those payloads are called Flight packages. RSC only ever renders on the server, either at build time (prerender) or at request time (dynamic). The client never renders RSC, it only decodes the payload. SSG and SSR are the processes of turning that payload into HTML.

### Restrictions

RSC comes with a bunch of restrictions. It is a one-and-done process that runs on the server, with its goal being to create those Flight packages.

#### No state

Having state means having the ability to change it. You can't rerender from just a Flight package. To recompute it we would need the source code, which is exactly what we want to avoid shipping by using RSC.

#### No side effects

We want neither state nor the ability to change through side effects. The client should reconstruct exactly what the server had previously encoded.

**Note:** This is also why you can't use non-deterministic functions like `Math.random()` in RSC.

#### Only serializable values

Like with side effects: if we can't reconstruct it on the client, we can't use it in RSC. JS does not allow us to serialize functions (unlike e.g. Clojure). We can use functions in RSC, but we can't pass them as props from server to client.

**Note:** We can reference functions that exist in the source code and use them on both the server and the client. However, that is different from passing a JS object from server to client.

### Structure

RSC gives us more power, better performance and makes our bundles smaller. However, avoid making your code unnecessarily complex just to save a few components from being shipped to the client. Obviously the majority of your page should use RSC, but you don't have to use it everywhere & always.

### Boundaries

When you put `"use client"` atop a file, you're saying everything past this point is on the client. Here is an example:

```tsx
// foo.tsx
"use client";

import { Bar } from "./bar";

export const Foo = (args) => <Bar {...args} />;

// bar.tsx
export const Bar = (args) => <div {...args} />;
```

Despite `bar.tsx` not being marked as a client component, it will still be in the bundle shipped to the client, because `foo.tsx` was marked as a client boundary (and it imported `bar.tsx`).

It is important to understand that `"use client"` does not mark a component as a client _component_. Rather, it marks it as a client _boundary_, meaning everything past that point will make it to the client. Think of `"use client"` as jumping out of the plane: there is no going back!

### Streaming

RSC enables Next.js to render an initial (non-blocking) view and stream in a more complete interface afterwards. This improves UX, as it allows us to show data faster. This is because we take pressure off the initial render (it needs to render less) and because the individual pieces of the follow-up can stream in independently of each other.

Example: We have an analytics page with a page-specific hero.

- Navigation: Hot from other requests, part of the initial HTML, shown instantly.
- Hero: Shows a skeleton in the HTML, loads from the CMS, streams in within a few milliseconds.
- Datatable: Shows a skeleton in the HTML, takes multiple seconds to load & compute on the server (afterwards it's in the server cache!)

Instead of having to wait multiple seconds for the HTML page to be ready, we saw it instantly. Loading skeletons kept the page from shifting and the content streamed in as soon as it was ready. Nothing was blocking, everything came to us as fast as possible.

As Datatable hints at: doing this work on the server also allows us to cache the operation, meaning a subsequent load of the page will be instant, as all the assets have resolved and can be turned into a static page. We will learn more about this in the [Caching](#caching) chapter.

Another big benefit is composition! Hero & Datatable began their requests in parallel and they both resolved independently of each other. Additionally, React lets us choose where we want to deal with the loading state, by letting us place Suspense where we want. We could wrap both components in a single Suspense and have them block until both are ready, or wrap each of them in its own Suspense and have them stream in independently (without blocking each other). React also does some smoothing under the hood to make sure there aren't too many renders & shifts happening at the same time as all the streams resolve. This smoothing is a couple hundred milliseconds of delay in Suspense (often a topic of ridicule from developers who don't understand performance bottlenecks in web applications).

### Leaks

Shipping less code to the client is a major feature of RSC. However, Next.js (when used with Turbopack) has a leak!

Here is an example:

```
- A (Server)
  - B (Server)
  - C (Server)
  - D (Server)
    - E (Client)
  - F (Server)
    - G (Client)
```

Component A uses B, C, D or F through a switch case, like so:

```tsx
export const A = ({ type }: { type: string }) => {
  switch (type) {
    case "B":
      return <B />;
    case "C":
      return <C />;
    case "D":
      return <D />;
    case "F":
      return <F />;
  }
};
```

Components D and F are the only components that import a client component (E, G). Those client components are also not used anywhere else.

**Here is the leak:** Even if the call to A didn't result in D or F being rendered, the bundle will still include E and G!

The way around this is to mark A as a client component and have it dynamically import all components using the [dynamic()](https://nextjs.org/docs/app/guides/lazy-loading#nextdynamic) function. This workaround, however, turns all of those in-between components (B-F) into client components.

## Dynamic

The server part of Next.js categorizes parts (and whole pages) as static or dynamic. If your page requires things that depend on the request, it will mark those parts as dynamic. If a page only has static parts, it will be static.

Dynamic parts include headers, cookies and URL search params. Smaller things like `Math.random()` or `Date.now()` can also cause issues, as those will be different for each request.

For marketing pages you should try your absolute hardest to make sure all (regular) pages are static!

Example:

- `/blog/guides/how-to-buy-paintings?utm_campaign=adsense2027` should be static
- `/search/cars?make=mini&model=cooper&transmission=manual` can be either static or dynamic (depends on customer needs)

Partial Prerendering made this less important, as a page can now be split into dynamic and static parts. However, you should have a really good reason to use it!

### Dealing with invariants

Dynamic parts aren't exclusive to functions that run on the server (e.g. cookies). Hooks can also cause issues (e.g. `useSearchParams`). Next.js does not pass the search params to this hook when it prerenders on the server.

In fact, it doesn't render that part at all. It forces you to make a decision instead.

Example: Imagine it did render it. It could either use the request's search params, deopting the page to dynamic. Or it could render with zero search params on the server and then hydrate on the client with the actual params, causing a mismatch.

Next.js does neither. Instead, it forces you to handle this invariant by making you place a Suspense boundary. That way the server renders the fallback and the client renders whatever the Suspense is wrapping.

This is an important part to understand. The server can (proxy shenanigans aside) only prerender one version of a page. Next.js does not allow you to prerender pages with specific URL params. As such, we need to use Suspense to take care of the disconnect between what the server prerendered & what the client should show.

**Recommendation:** Take the `useSearchParams` call out of the component and pass its normalized contents to the component. This way you can use the actual component as the fallback in the Suspense.

Example:

```tsx
const Page = () => (
  <Suspense fallback={<FooContent />}>
    <Foo />
  </Suspense>
);

const FooContent = ({ search }: { search?: string }) => <p>Search: {search}</p>;

const Foo = () => {
  const s = useSearchParams();
  return <FooContent search={s.get("search") ?? undefined} />;
};
```

### Edges

RSC is easy to misunderstand and its implications go beyond the usual state & rendering mental model for React.

One of the biggest oddities in Next.js and in RSC is the `not-found.tsx`. We have learned that RSC is the process of turning React into a Flight payload. We have also learned that we (almost) always use RSC in combination with SSG or SSR, so we have some HTML to display to the user before React has a chance to reconcile the RSC payload on the client.

However, all of this gets challenged in the `not-found.tsx`! See for yourself: open any 404 page in an app directory application **with JavaScript turned off**. You will see three things happen:

1. The server will spring to life and do work on the not-found page
2. The page will be blank (where the `not-found.tsx` slot should be)
3. The HTML contains the RSC for the not-found page

So we have the server generating the RSC, but it isn't server-side rendering HTML for it (like with a page). Instead, it includes the RSC packages in the initial response and waits for the client to reconcile them. Even more bizarre: it does this even for not-found pages that have been cached as 404 in the CDN!

We essentially have client-side rendered RSC here. There is no workaround, although this is hardly a problem.

A related, more expensive oddity: a root `app/not-found.tsx` (i.e. top-level) gets embedded in the RSC payload of **every** page. We have seen this add a second `<html>` tree of ~40 KB to every page. To fix this, use a `global-not-found.tsx` (instead of a top-level `not-found.tsx`).

## Caching

Cache Components made everything way easier and way more powerful. You should use them! Much like Suspense, they allow you to communicate your intent to Next.js. The app directory can then use those hints to know what needs to change when, and to yell at you when you create a bad user experience.

We have three types of cache interactions:

- cache forever
- cache for some time
- never cache

### Cache life cycle

You can control the shelf life of a cache by attaching a cache profile with `cacheLife`. Next.js ships with some cache profiles included, `max` being the biggest (revalidates after 30 days, expires after 1 year, so not forever). We define our own `forever` profile in `next.config.ts` (revalidate/expire 365d) and use that for CMS content that is revalidated on-demand.

However, arguably more important is which cache you're hitting. Next.js has three cache directives: `"use cache"`, `"use cache: remote"` & `"use cache: private"`. `"use cache"` caches in the memory of the server. If run on serverless, the teardown of the lambdas clears the cache. `"use cache: remote"` uses platform-specific caches to make sure the cache stays around. On Vercel this is the Runtime Cache, which is per-region but persists (you can watch it under Observability > Runtime Cache). Remote adds a bit of latency to every call. `"use cache: private"` is never stored on the server and is meant for per-user data.

What worked for us: plain `"use cache"` on pages and components, as a signal to Next.js, and `"use cache: remote"` on the data fetchers.

**Note:** Neither cache persists through a deployment, as the build ID is part of the cache key.

**Gotcha:** `cacheTag` with a Promise (missing `await`) silently tags nothing.

### Tag association

Included in Cache Components is a mechanism that associates caches with the pages that they're called from. If you have `getHero(id)` and `/blog` calls it with id `123`, Next.js knows that it needs to rebuild the blog page if you ever revalidate the tag associated with `getHero(123)`. This means the happy path for a CMS revalidation is to tag every fetch with the CMS entry ID and call `revalidateTag("123", "max")`.

**Note:** In practice it is rarely just the one ID that we need. E.g. a URL that 404'd and later gets content won't be fixed by tag revalidation. Use `revalidatePath` (or a path/slug tag) for that.

### Tag inheritance

If cache component A calls cache component B, A will inherit all of the tags from B.

### Tag limits

A single `cacheTag()` call accepts up to 128 tags, each at most 256 characters. Anything beyond that is dropped with a console warning, so keep an eye on the build logs. A page can contain multiple cache entries, so the limit per page is higher than 128.

### Lifetime inheritance

This is a bit tricky. Let's assume cache component A calls cache component B.

- If A sets a life, it always uses its own, whether it is longer or shorter than B's. B's shorter life is "trapped" inside A.
- If A does not set a life, it uses the `default` profile (15 min revalidate). A shorter life in B (e.g. 5 min) reduces A's. A longer life in B (e.g. 8h) does **not** extend A, it stays at 15 min.
- If A does not set a life and B is short-lived (zero revalidate or expire under 5 min), Next.js throws during prerendering.

## Server Actions

Server actions are not for fetching data. An _action_ is not a _query_, it is the response to intent: we want to transmit something to the server. Nothing stops Next.js from widening its implementation of React Server Functions to also be viable for fetching (like TanStack Start did), but the team has so far not done anything about it.

You should not use server actions for fetching because:

- they don't cache (they're POST requests & skip the Vercel CDN cache)
- they can't run in parallel (one running server action causes all further server actions to queue)
- there isn't a great surface to consume them (this is intended)

There is an exception to this: you can return RSC from a server action and React can reconcile it natively. This is really helpful if you don't want to ship the JS required to render the components! Example: an infinite scroller with dozens of complicated components. Instead of shipping them in the bundle, you can offload them to the server. This theoretically also allows you to generate novel UIs on the server, where the blueprint wasn't known at the time of bundling the application.

Notice how these exceptions are incredibly niche and don't apply to marketing pages.

**Note:** Think of server actions as something that got added so you don't have to turn server components into client components just to do simple forms etc.

## Partial Prerendering

PPR is a game changer. However, we should be cautious where we apply it. For marketing sites, most of our tasks deal with static scenarios, and the performance (and cost) downsides of dynamic work don't vanish just because we use PPR. Its biggest downside is how complicated it is. Depending on how the cache life is set up around the dynamic parts, a lot changes.

Examples:

- Parent has long cache life + dynamic part has short cache life = Dynamic part might get swallowed!
- Parent has no cache life + dynamic part has long cache life = Dynamic part will get optimized into the static shell with a long cache life

These all make sense if you think them through, but they're easy to miss when just building.

## Route Relations

The app router gives you lots of pieces to build exactly what you want to build. Some of the pieces include `layout.tsx`, `page.tsx`, `template.tsx`, `not-found.tsx`, `error.tsx`, `loading.tsx`, parallel routes, intercepted routes and route groups. Some relations between these pieces are obvious. A page always relates to the layouts above (and beside) it. Same with templates, parallel routes & intercepted routes. Less well known is the relationship between a page and the other page-components for that slot: loading, error, not-found (among others). Obviously page and loading are connected: the loading component needs to be shown until the page is ready. That relationship is also trivial, as you can't await in a `loading.tsx`.

However, less well known is the relationship between page and not-found. Next.js renders both page & not-found essentially in parallel, to reduce wait time if a `page.tsx` suddenly bails out to the not-found via `notFound()`. This has the strange side effect that all tags that got used in the rendering of the not-found are also attached to the `page.tsx` (and vice versa). So if your not-found uses a special tag, that tag will be attached to all `page.tsx` in the same slot. If you ever revalidate that tag, all `page.tsx` at that slot also become stale.

The same applies to `forbidden.tsx` and `unauthorized.tsx`. `error.tsx` is excluded, as it is a client component.

This makes attaching tags to `not-found.tsx` oddly expensive. If negotiable: try to keep the `not-found.tsx` completely hardcoded. If not (e.g. a CMS-driven 404 with site chrome), give it its own `forever` cache and a dedicated tag (e.g. `"404"`) that your webhook revalidates explicitly.

`not-found.tsx` is resolved by its position in the app directory, not by URL! An optional catch-all like `[[...slug]]` needs its own `not-found.tsx` (re-exporting the root one is enough).

**Note:** Next.js 16.3 added [`next/root-params`](https://nextjs.org/docs/app/api-reference/functions/next-root-params), which allows you to retrieve e.g. a `[locale]` from the route in a `not-found.tsx` via `import { locale } from "next/root-params"; await locale()`. This is helpful because `not-found.tsx` receives no arguments (unlike e.g. `layout.tsx`). Alternatively, use static `app/de` & `app/en` trees with a hardcoded locale each.

## A word on proxy

Treat the proxy as an escape hatch to handle whatever the app directory (or pages directory!) is incapable of providing. It should be your last resort. You can only define one proxy, and any addition has to be done by expanding the matcher. In other words, any addition degrades performance (and increases cost) as you opt more and more of your application into the proxy.

There is nothing wrong with the proxy. It is, however, often misused for things that can be achieved with other (more performant) tools (e.g. rewrites). Lots of tutorials (or third-party integrations) will point you to the proxy, so they don't need to explain more of the setup (or handle edge cases).

## Redirects

Before reaching for a proxy, you should ask yourself if this can be done via a redirect (or rewrite). Redirects give you a lot of freedom. You can even add multiple for the same source and then decide based on headers:

```json
{
  "source": "/app-stores",
  "has": [
    { "type": "header", "key": "user-agent", "value": ".*(iPhone|iPad|iPod).*" }
  ],
  "destination": "https://apps.apple.com/de/app/cr%C3%A9dit-agricole-deutschland/id6766425004",
  "statusCode": 307
}
```

## Rewrites

Rewrites are incredibly powerful! See them as an extension to the app directory: the app folder stays optimized to work as performantly and flexibly as possible, while rewrites let you make the resulting URL as pretty as you need it.

Example: Rewriting `.md` URLs to conform to the internal structure, where the markdown versions live under `/md/[[...slug]]` instead:

```json
{
  "source": "/:path(.+)\\.md",
  "destination": "/md/:path"
}
```

Rewrites also allow you to match on headers, like redirects:

```json
{
  "source": "/:path*",
  "destination": "/md/:path*",
  "has": [
    {
      "type": "header",
      "key": "Accept",
      "value": "text/markdown"
    }
  ]
}
```

**Note:** `/:path(.+)\\.md` does not match `/index.md`, so the root needs its own rewrite.

Rewrites are resolved on the server, inside the same request (unlike redirects, which cost an extra hop: 307, then a second request). However, the App Router client knows nothing about your rewrites. Usually the client can predict the route of e.g. `/blog/post-2` from an already prefetched `/blog/post-1` and skip asking the server. For rewritten URLs that prediction is disabled, so every rewritten URL needs its own round-trip to the server to resolve its route. With `<Link>` prefetching this mostly happens before the click, so it is a small price to pay for the freedom that rewrites bring.

## Incremental Static Regeneration

Next.js has great revalidation support: both on-demand through `revalidateTag` & `revalidatePath`, and time-based revalidation through ISR. You should use on-demand revalidation if possible. However, there are reasons to pick time-based revalidation instead:

1. Resolving page dependencies is too difficult/expensive
2. Way too many pages to resolve in adequate time
3. Dependencies are such that every update would hit every page anyway

Revalidation remains the hardest part of working on marketing pages, so it is good to have ISR as a backup. However, ISR generates significantly more cost than on-demand revalidation! Think **multiple** orders of magnitude more.

## Activity

Cache Components come bundled with Partial Prerendering & Activity. This is a major source of pain! With Cache Components turned on, Next.js uses Activity to keep the last few routes alive (hidden with `display: none`) instead of unmounting them. Effect cleanups run when the route is hidden, but the DOM and state stay around and get revealed again on back navigation. Third-party integrations that don't survive this cleanup/re-setup cycle can easily explode and maybe even kill the entire browser tab.

Old state (e.g. open burger menus) and view transitions are also likely candidates for failure.

These cases are difficult to find, as you need to not just navigate away, but navigate back again. On faster machines some issues might not even surface (while slower machines suffer massively). If you have third-party integrations that are more than an iframe (maps, players, forms), make sure to navigate back and forth and navigate in loops a couple of times to be sure that they don't run into issues.

We have identified Mapbox & Vidstack to be particularly vulnerable to these issues.

You can defuse those bombs by strategically placing a key on components that use those problematic integrations:

- `bfcacheId` from `useRouter()` resets the subtree on push/replace navigations but still restores on browser back/forward
- `usePathname()` resets whenever the path changes
- combine both if needed: ``key={`${bfcacheId}-${pathname}`}``

Don't try to force remounts with `useEffect` cleanups! The Next.js docs call `bfcacheId` a migration tool and recommend per-pattern resets (see the "Preserving UI state" guide), but for third-party integrations the key is the pragmatic fix.

## Sitemap

While being a nice addition, the `sitemap.ts` can be cumbersome to deal with. First of all, Google reads at most 50,000 URLs (or 50 MB) from a single sitemap. If you need more, split it up with `generateSitemaps`. Another issue is that with Cache Components on Vercel, the sitemap is stored as a static asset that does not get revalidated via cache invalidation (like a `page.tsx` does)! This makes a static `sitemap.ts` useless for a CMS-driven project. To get around this, you have two options: turn your sitemap into a `route.ts`, or call `await connection()` inside your `sitemap.ts` (to make it dynamic). Neither is ideal, but both get the job done.

## Suspense

We have already mentioned Suspense a lot. Both React & Next.js use it as a solution to many problems. For static sites there is one additional detail to keep in mind: the initial HTML only includes the fallback. This is somewhat obvious, but important to keep in mind.

Imagine you have to call `useSearchParams()` far up the tree. You need to wrap the entire tree in Suspense (to keep the page static). Without a fallback, everything inside the Suspense will render as an empty void in the HTML sent to the client. Adding a fallback fixes this, but that puts a lot of pressure on hydration (and thus your main thread)!

**Recommendation:** Suspense will make it work, but you should question whether this is the right way before you even get there.

## Route handlers

Next.js makes it easy to spin up & co-locate API routes. They work reliably, are purposefully kept simple and give you everything you need. There is one issue, however: unlike pages, they're dynamic by default.

**Disclaimer:** This section is based on our experience and might be outdated. The Next.js 16 docs claim that with Cache Components, `GET` handlers prerender like pages as long as they don't touch runtime data.

Example: Let's say you have a dynamic segment like `/api/products/[id]/route.ts`. If we used a `page.tsx` like `/api/products/[id]/page.tsx`, we would expect Next.js to use the Vercel CDN to cache the response, as long as we don't use any dynamic functions. While this is true for pages, it is not true for API routes!

In our experience API routes are dynamic. Each request against them will incur compute. If you deploy on Vercel, you can work around this by setting a `Cache-Control` header like this:

```ts
export async function GET() {
  return Response.json(
    { message: "Hello" },
    {
      headers: {
        "Cache-Control": "public, s-maxage=60, stale-while-revalidate=300",
      },
    },
  );
}
```

However, this caches it in the Vercel CDN, and functions like `revalidateTag` or `revalidatePath` won't invalidate that cache!

This is unfortunate. If you want to stay in the Next.js ecosystem, you're better off moving the body of the request (i.e. the actual work) into a cache component and eating the compute cost on every request.

## Bots

Bots that receive the fully blocking HTML (instead of the streamed version) can cause cache `BYPASS`es and with that unwanted API calls. Set `htmlLimitedBots` in `next.config.ts` to a regex that matches (practically) nothing to keep that list minimal.

## Draft mode

The Next.js team seems unsure if `draftMode()` should be a dynamic function. Internally it reads from a cookie, so technically it should be a dynamic function. However, the cookie is always the same and the reason for `draftMode()` to exist goes against the idea of making it dynamic.

As of right now it is not a dynamic function. It will not deopt your static page into dynamic.

Because of this and the oddities with `not-found.tsx`, we agreed not to use draft mode in `not-found.tsx`.
