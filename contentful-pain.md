---
name: contentful-pain
description: When you're making architectural decisions for a Contentful project
disable-model-invocation: false
metadata:
  version: "1.1.0"
---

# Contentful pain

Contentful is a popular CMS that has a solid type system and offers both a REST & a GraphQL API. It is incredibly expensive, which brings with it an expectation of quality. However, reality often falls short.

## Types

Contentful Types are just containers for whatever you need them to be. Pages, Links, Products, Categories, Person - it all works and leads to a simple mental model.

Tip: Each entry has a "display field", which is the field that will be shown in the Contentful search for that entry; it can be difficult to find a field that sums up the entry (e.g. what if you split up first and last name for Person). Historically we have just added a `name` field to each content-type to keep the entire system consistent.

### Relations

Relations are one of the most important features in Contentful. Not only do they make our job as developers much easier (e.g. "entry id" to "page slugs" is just a walk up the relations tree) but they also make it dramatically easier for editors to reason about what any given page might include. Defining connections in code ("this component _just links_ to our configurator") seems like a good idea, but it has downstream effects on fetching, revalidation, preview and the editor's confidence, as well as the communication effort required.

We have historically favored attaching as much as possible. Example: Navigation & Footer need to be configurable (instead of being auto-generated). As such, we usually attach navigation & footer to the page. That is totally unnecessary from the editors' perspective, but it makes it dead easy to find **the right navigation for a specific page** (vs. just requesting "any" or "the first" navigation) from a developer's perspective. Having everything a page needs attached to that page also shrinks the mental model for the editor, as they know that they will find everything within that page entry.

Note: This can also be a downside. 404 or error pages aren't really pages you set up in Contentful, yet they would then need a navigation & footer. See "Global configs" section for more.

### Global configs

Many CMS projects at some point opt to introduce a global config, which allows configuring project-wide settings. Examples include: Copyright, footer, homepage, social media links. Things that are relevant for all pages, but are difficult to link to. This isn't specific to Contentful (we have observed this pattern in e.g. Storyblok, too).

**Recommendation:** Our opinion on this is neutral. We strive towards making relations as clear as possible (because of the downstream effects) and if a global config can help with that, go for it. However, it should only be introduced if it really is necessary. Things like copyright, homepage and social media links should just be hardcoded. Special elements on the page like a link to a configurator, should just be links that an editor can configure. However, navigation & footer (if they should be the same for every page) are a great fit. Another good example are announcements that need to be present on every page.

### Pages

While relations are very important, you should never under any circumstances generate page hierarchies via relations in Contentful.

Example of what you should not do:

- Page Baz: Slug `/baz`, no children
- Page Bar: Slug `/bar`, has `Baz` as child
- Page Foo: Slug `/foo`, has `Bar` as child

Resulting in these URLs:

1. `/foo`
2. `/foo/bar`
3. `/foo/bar/baz`

This seems logical at first, however, resolving links becomes a nightmare! Imagine you have a `Link > Internal` component which takes a `Page` as input. This is awesome for the editor! However, the API will only give you the slug of that page (i.e. `baz`) and the id of that page. It is now up to you to resolve where that should link to or where that page should be rendered. This leads to you needing a whole system that just works on resolving these pages.

This will also bloat your normalization layer, as almost every normalization function now needs a page tree variable passed to it, in case somewhere deep it would need to resolve a link. It is also trivial to accidentally ship this huge JSON of the page tree to the client, slowing your page down (for no reason).

This also makes it difficult to reason about the page tree, as Contentful only ever shows the slug, not the full URL for each page. It is also easy to make a mistake and accidentally render the same page in two places (without ever noticing). Even worse: Introducing localization makes every con hit even harder.

**Recommendation:** Each page has its full slug! To revisit our example:

- Page Baz: Slug `/foo/bar/baz`
- Page Bar: Slug `/foo/bar`
- Page Foo: Slug `/foo`

This is a bit more work for editors, but it forces them to make a decision, making it easy to spot incorrect data and wrong behavior! It also makes the code trivial.

Tip: I am calling it "full slug" instead of URL. Editors should never need to enter the domain, a port or other important parts of your URL (e.g. the locale).

### Conditions

Contentful does not support conditional fields. There are apps for it, but we have never tried them. There is no support via the API. You can build your own app.

**Recommendation:** Do not build your own app.

### Singletons

The type system is quite limited. Singletons are not explicitly supported. This can cause issues, as many marketing sites (and other types of projects) often have things that should only exist once. This makes fetching brittle, as you either rely on the fact that no editor ever accidentally creates another entry of that Content-Type or you try to hack together a solution using tags.

You can actually use the type system to emulate singletons by doing the following:

1. Add a symbol field
2. Only allow a single string via validation
3. Make that string the default value
4. Set the field to be unique
5. Hide the field

Editors will now not be able to publish another entry of that content-type.

Note: Doing this through the UI means you have to save the field + content-type multiple times, as the UI does not allow you to do this in one go.

**Recommendation:** Don't do this.

### Optimizing for existence

Content-Types are incredibly expensive. Bumps in packages can cost as much as a used car. This means every project needs to be strategic with its content-types. Some customers are able to spend more; for those you can split what should've been a single content-type with conditional fields into two content-types (e.g. `Car > ICE` & `Car > EV`). For customers that can't afford to dump the equivalent of a used VW Golf per year into Contentful, you have to just be pragmatic and combine the fields into a single content-type (e.g. just `Car`), with lots of optional fields (but with normalization that aggressively exits early). Work with the editors and be empathetic, but also build up a reputation so they can be understanding of the situation.

## Localization

Contentful allows you to localize content. The UI is intuitive and the API works well, however there are a couple of things that can cause issues.

### Localizing content

Contentful is very rigid in how it allows you to localize content. You choose what to localize on a per-field basis. Each field inherits all locales. This means you can't apply only a subset of locales to a specific field. Locales can be languages (e.g. `en`, `fr`, `de`), countries (e.g. `de-DE`, `fr-CA`) or something you have made up (via CMA).

The localization system gives you two escape hatches in the form of fallback and optional languages, however treat these as an absolute last resort!

Contentful mentions [four localization strategies](https://www.contentful.com/help/localization/field-and-entry-localization/#richtxt-localization-approaches), although only the first two can realistically be used. Both have their downsides.

**Recommendation:** Try your hardest to not mix and match these strategies! While their benefits interplay nicely, their downsides can cause massive headaches for developers and editors.

#### Field-localization

What you expect. E.g. you're a car manufacturer and have a `Car` content-type where each _attribute_ is localizable. So the name, the trim level, the description, maybe some legal text. You could also do things like "available colors" (in case the car is not sold in a specific color in a specific country). To expand this example, let's say you also sell used cars and have the `Used Car` content-type for that. Here the VIN field would not be localizable, as it is an attribute that does not change between languages (it is unique to the car). Additionally, you wouldn't localize something like the actual color of the car or the transmission; these are again essential attributes of the car.

It is tempting to make references localizable as well, but it is highly recommended not to do that with this strategy! This causes confusion and makes the number of available fields explode.

**Summary:** This approach works best if the majority of the editing time is spent on the content vs. the structure. Editors can click on one entry and supply all the attributes they need on that one entry. This also means they don't have to switch between multiple entries if they need to update a field later. **However**, this means the structure of the website needs to be recreated multiple times. Each page needs to be created for every locale. Each in-between entry that only exists to orchestrate the page needs to be created for every locale.

In short: each attribute update happens in one entry, each orchestration update happens in many entries.

#### Entry-localization

The inverse of the previous strategy. Instead of making the leaf entries localizable, we make all the branches in between localizable. Instead of making a single entry of a `Car`, you have many, with the language mentioned somewhere. The places that reference this entry are localized. This means you have the structure in a single entry vs. in many. The downside is that you have the content in multiple entries (vs. in a single one).

**Summary:** This approach works best if the editors spend more time on the structure. Little content (or content that does not change often) but dozens or hundreds of pages in many languages.

### Fallbacks

Contentful is pretty unique in that it lets you set a fallback per locale. E.g. you want to reuse structures already set in your base language English, so you set `EN` as the fallback locale for German, Spanish, French and Japanese. Requesting content in `DE` on a field that is empty now returns the content for locale `EN`. The UI also indicates this relation. This can take a lot of pressure off of small teams that need to update content for many languages. However, treat this option as a massive footgun! It makes it trivial to silently turn websites into a patchwork of different languages! Most egregiously: It introduces the idea to the customer that, with enough magic, they will not need to touch their content! This is a dream that will turn your life into a nightmare.

This feature can be legitimately helpful for teams that know the structure of their page is almost identical across multiple languages. However, this option is global, for every localized field in the entire environment! This means you are likely to activate it for a field where you don't want it (probably without knowing)!

**Recommendation:** Try to avoid fallback. Do not use fallbacks if you're doing localization by content, unless within a language (e.g. reuse `German` for `German - Germany` and `German - Switzerland`). Using fallback to reuse an existing structure is okay, but has historically been confusing to editors and caused a lot of communication overhead to help editors figure out where their content is.

### Allowed to be empty

This goes hand-in-hand with fallback. You can opt an entire language into being optional. You can't decide on a per-field basis; this applies to every localized field in the entire environment. Like fallback, this will cause fields you don't want to be optional to become optional. Unlike fallback, this can become an issue because you will receive nothing where you'd expect to receive something (i.e. breaking the contract you have built up). As such, like fallback, it is a massive footgun. However, it almost always has to be used together with fallback.

**Recommendation:** Try to avoid.

## APIs

Contentful offers many APIs. Here is a quick overview:

- CDA: REST API for content
- GQL: GraphQL API for content
- CPA: REST API for preview content
- CMA: REST API to do writes on the content or content system

Additionally, Contentful also allows you to optimize images (quality, format, width, height) through their Images API. Important: The Images API quietly dies if you try to optimize an image that is too big (>20MB, >4000px in width or height), so make sure the editors know how to use Contentful as a DAM.

### GraphQL

We prefer the GraphQL API over the REST API because it is easier to infer the return type, makes code more reusable & because it provides snappier Live Updates in the Live Preview.

#### Complexity

The biggest issue with the GraphQL API is the query complexity. Instead of specifying a max depth, Contentful allows a max complexity of 11,000 per query, which is computed by the fields you specify. A single field like `title` only has a complexity of `1`. If your query requests the fields `title` and `name` (both simple fields) you end up with a complexity of `2`. Collections (e.g. cards in a cards module) act as multipliers. Requesting a collection of a type that has `title` defaults to 100. This means setting limits is very important! If you know you will only need e.g. a max of 50, always set that limit!

If you reach the complexity limit, you will have to remove parts of your nested structure and move those into separate queries that run afterwards in another request.

Tip: If you have to split your queries, target the collection of the type and use the `where: { x: { x_in: $yourVariable } }` filter to get all entries in one go (assuming n<1000).

```graphql
        query ModuleAccordionItemsQuery($ids: [String]!, $limit: Int!, $preview: Boolean) {
            moduleAccordionItemCollection(
                where: { sys: { id_in: $ids } }
                limit: $limit
                preview: $preview
            ) {
```

#### Phantom results

In the GraphQL API you can set `preview: true` to receive unpublished changes. The GraphQL API also lets you set this independently on every reference you request. However, the GraphQL API has a quirk where even if you set `preview` to `false`, you will still receive unpublished references as stubs. This is very likely to lead to issues in your application. You can guard against this by checking if an id exists for the reference:

```graphql
fragment NavigationFragment on Navigation {
  primaryCta(where: { sys: { id_exists: true } }) {
    ...LinkFragment
  }
}
```

#### Collection Naming Clash

The GraphQL schema is generated by Contentful based on our Content-Types. Each type receives at least two queries.

Let's say we have a type called "Foo". We would get the following queries:

1. `Foo`: getting a `Foo` entry by id
2. `FooCollection`: getting many `Foo` entries

This becomes an issue if you also end your Content-Types with `Collection`. Let's assume we have the two content-types `Bar` & `BarCollection`. This would cause a name clash that we can't resolve; you would end up with **three** queries for these **two** content-types (instead of four)!

1. Bar
2. BarCollection
3. BarCollectionCollection

Thus, avoid ending your content-types in `Collection` at all costs!

#### RTEs

Rich text fields have four major problems:

1. You cannot use GraphQL Fragments, as the contents of each RTE are scoped to the field; what you receive back isn't `RTE` but rather `FooRTE`, so no Fragment will fit (unless on the same field)
2. The schema does not honor validation, so the GraphQL schema allows you to request content that the content-type does not allow; e.g. you can retrieve embedded entries even if they're forbidden by the validation on the field
3. Contentful hardcodes the limit for references to 100 and you cannot change it, so if you retrieve embedded entries, you will incur the complexity of 100, massively bloating your query
4. TypeScript type support for GraphQL RTEs is incredibly poor from Contentful; you will have to build your own system

There are no workarounds for any of these, you just have to live with them.

**Note**: The Contentful UI does let you specify an upper bound for embeds (links, entries, assets) - it remains to be seen if this affects (positive) query complexity.

##### Phantom results in RTEs

**Important**: The phantom results also affect RTEs, however there they're a much bigger problem! Contentful gives you no way to filter them out ([working as intended](https://support.contentful.com/hc/en-us/articles/9904387047837-How-to-remove-UNRESOLVABLE-LINK-error-GraphQL?input_string=graphql+api+returns+error+for+unpublished+embedded+entries+in+rte)), so you can't rely on the `id_exists: true` trick. This means if somebody references an unpublished entry in your RTE, that query will always throw errors. We're working around this by wrapping queries that use an RTE with a function that skips `UNRESOLVABLE_LINK` errors:

```ts
export const shouldSkipError = (error: CombinedError | null | undefined) => {
  if (!error) {
    return true;
  }

  if (!error.graphQLErrors) {
    return false;
  }

  return error.graphQLErrors.every((e) => {
    const ctf = e.extensions?.contentful;

    if (!ctf || typeof ctf !== "object" || !("code" in ctf)) {
      return false;
    }

    return ctf.code === "UNRESOLVABLE_LINK";
  });
};
```

### REST

When compared to the GraphQL API, the REST API allows more data to be retrieved with a single API call, however it has historically been slower when doing Live Updates in the Live Preview.

#### Depth limit

The biggest con of the REST API is the depth limit, handled via the `include` parameter. Instead of calculating the complexity of your query, the REST API returns all content up to a depth limit that you set. The API tops out at `10`, which is incredibly generous when compared with the GraphQL API, however the depth limit causes two major issues:

1. If you don't get what you want, it usually is half cut off, making debugging really strange; historically this made return types really weird too, but I suspect they have fixed this
2. You can run into infinite loops when working with the data

You almost always hit issue #2, so let me expand on that. Let's assume you have a page with a module that contains a link to the original page. If your `include` is big enough, you have created an infinite loop through the link back to the original page. Trying to ingest this JSON as-is will cause a recursion error. The simplest way to break this is to use the `stringifySafe()` method:

```ts
const data = await client.getEntries({ content_type: "page", include: 2 });

const plain = JSON.parse(data.stringifySafe());
```

#### Unpublished content

Unlike the GraphQL API, retrieving unpublished (i.e. draft) content isn't a simple toggle but requires a different URL (and a different token).

```ts
export const getRestClient = (preview?: boolean) => {
  const accessToken = getAccessToken(preview);

  return createClient({
    space: process.env.CONTENTFUL_SPACE_ID ?? "",
    environment: process.env.CONTENTFUL_ENVIRONMENT_ID,
    accessToken,
    host: preview ? "preview.contentful.com" : "cdn.contentful.com",
  });
};
```

Unlike the GraphQL API, it does not suffer from phantom entries.

## Preview

Setting up the Contentful Live Preview is easier than it looks. For Contentful, when we talk about the Live Preview we usually mean these three distinct features:

1. Live Preview
2. Live Updates
3. Inspector Mode

Live Updates and the Inspector Mode are features that work within the Live Preview. They massively improve UX, however Live Updates force you to make a lot of architectural decisions to make them work correctly.

### Reaching the preview

In our Next.js projects we generally try to reuse the existing page generation structure to cut down on code and minimize differences between how something works in the preview and how it works once published. However, you can also build a dedicated page that is just for the preview. Contentful also allows you to specify links for each content-type, so you can even build a bespoke preview for each content-type.

Here is what happens in our Next.js projects when a user clicks "Open preview" in Contentful:

- Page does request against an API Route with the id of the entry
- API route turns on draft mode
- We check on which page the requested entry is on
- API route redirects to that page (with the draft mode turned on)
- Next.js sees draft mode, ignores ISR cache & does a bespoke rendering of the page
- All API calls see that draft mode is on and request preview data (instead of published data)

### Draft Mode

In Next.js we check [draftMode()](https://nextjs.org/docs/app/api-reference/functions/draft-mode) to determine whether we're in the preview.

Note: The Next.js team seems conflicted on whether this should be a dynamic function. The Next.js documentation had it marked as one for some time, however as of writing this, the docs make no mention of it. Under the hood draft mode checks if a cookie is set, which should mark it as a dynamic function (like [cookies()](https://nextjs.org/docs/app/api-reference/functions/cookies)). However, the Next.js team appears to have made a special exception for it. I consider it safe and see no reason not to use it.

Tip: `draftMode()` sets the cookie on the domain of the request. This can be disruptive for editors if you choose to use the same domain for the preview and the live site, as it will cause draft content to show even on the live site (i.e. outside of the preview). If you deploy to Vercel, every deployment receives a template `.vercel.app` domain, even production! So you can reach your live production site from both `foobar.com` and `foobar.vercel.app`. If you set the preview URL to be the `.vercel.app` one, you can avoid leaking draft content onto the live site for editors without any drawbacks.

### Architectural headaches

Choosing an entrypoint and setting up the individual pieces for the preview is simple, however orchestrating live updates while using as many React Server Components as possible is difficult. There is an inherent tension here:

1. You need to use a hook ([useContentfulLiveUpdates](https://github.com/contentful/live-preview#live-updates)) to receive live updates
2. You need to use React Server Components to fetch (the initial) data
3. You need to split up fetching in many places because Contentful does not allow you to get everything in one request (see API section)

So 1. and 2. are at odds, and any progress on those will cause 2. and 3. to be at odds. You can't use hooks in RSC, thus you need to be on the client, and you can't fetch from Contentful on the client (without workarounds). So you want to fetch more with the initial fetch, which you can't, so you cut up the queries into more pieces, requiring more hook calls which you can't do because of RSC.

Here is an example of how this tension will surface:

1. You have a page which has modules
2. You can't fetch all modules' content at once, so you fetch each independently
3. You want to enable live updates for the modules, so you branch after fetching module content (if we're in the preview) and route the data through `useContentfulLiveUpdates`

This leaves us with a structure where we have `Page -> Modules -> (Module -> ModuleLiveUpdate -> ActualModule)[]`.

There are two issues: Because we only fetch the page on the server, if an editor adds another module to the page while in the live preview, the preview would not show that new module, because that reactivity depends on `useContentfulLiveUpdates`, which we can't use on the Page/Modules level (because we're on the server there to fetch the page). Because everything past Module is behind a `useContentfulLiveUpdates` call, everything past that point also needs to be able to render on the client. This doesn't mean everything needs a `"use client"`, however it means we can't rely on those modules being RSC (e.g. fetch in them).

To fix the earlier issue of not showing changes in which modules should be rendered, we would need to move the `useContentfulLiveUpdates` up (i.e. to the left in our structure). This would mean more of our component tree would need to support being on the client, i.e. lose RSC support. This can be achieved: we can totally make a `ModulesPreview` component as an alternative to the regular `Modules`, where we use something like `useSWR` to call a proxy endpoint in our backend to retrieve the draft content of those modules, which we then feed into `useContentfulLiveUpdates` at that `Modules` level. At this point you will need to rebuild the entire page generation pipeline, as all elements that can lead to visual change need this attention (navigation, hero, breadcrumb, footer). The concerns here are that you will maintain two worlds (live & preview) and the preview will slow down, as you introduce a client-side fetch waterfall (fetch preview page -> fetch modules -> fetch module).

**Recommendation**: Get a feel for what the customer needs and can (financially) support. If they need a foolproof system that has them living inside of the preview 24/7 (and they have the budget for it), go nuts! Otherwise, be pragmatic. Contentful has already introduced an alternative to Live Preview called "Studio" and _word on the street_ is that they're already working on a replacement for that, so be aware of the possibility of the Live Preview being deprioritized in favor of a new UI in the distant future.

## Integrating into Next.js

### Structure

In our modern projects we follow a simple structure: fetch -> normalize -> render. Clarity is important because it shrinks the mental model and makes implementing the Live Preview & revalidation easier.

Example: In Next.js we technically could fetch inside of "render", i.e. React components, if we can make sure they're Server Components. However, all of our projects need to run in the Live Preview, which is client-side rendered. This means everything to the right of "fetch" needs to be able to run on the client. As such we have a dedicated fetch layer where all the requests are done. This has the benefit that our developers aren't really writing Next.js code, but rather just React (meaning they don't need to know about client/server boundaries).

### Caching

Next.js deduplicates fetch requests within a request context. The official Contentful SDK still uses Axios (AFAIK), so deduplication does not work there. If you're using GraphQL, you're probably using fetch, but you're probably also doing POST requests (instead of GET), so there is no deduplication here either. To deduplicate your requests, make sure to wrap the function that makes the outgoing call in `React.cache`.

This is important because you might fetch a resource in a `layout.tsx` and a `page.tsx` (or you have multiple parallel routes that rely on the same data as the original `page.tsx`).

More important are Cache Components! They greatly simplify the mental model. If you intend on using Cache Components make sure to use `"use cache: remote"` instead of just `"use cache"` for queries against Contentful.

### Revalidation

Implementing a solid revalidation mechanism isn't easy. The wrong architectural decisions can really cause a lot of pain here. As such, let me recap some of the important decisions:

1. Have a specific layer where you fetch & have the functions doing the fetching clearly highlighted as such and located in an easy-to-identify place
2. Set up your content model to use as many Contentful relations as possible, so dependencies to Contentful don't create edge-cases and can easily be spotted
3. Keep the number of components doing fetching as low as possible
4. Do not use nesting for pages (i.e. page has a children field where you place pages to establish a page tree)
5. Use Cache Components so Next.js can help you

Cache Components make revalidation a lot easier. Partial adoption will deliver the functional benefits of Cache Components, however to achieve simplicity you will have to fully commit to them! Adopting Cache Components can be painful, so make sure to read up on the changes.

**Note:** The revalidation strategy will be different for every project, as the content model is different for all of them! However, the core concept remains the same.

All approaches use a Webhook from Contentful that hits an API route to be informed about an update.

For your production webhook, make sure you're pointing at the final URL. Vercel recommends using the `www` subdomain, so if your website lives on `www.foo.bar`, setting the production webhook URL to just `foo.bar` will lead to Contentful abandoning the request! The URL will need to be `www.foo.bar`!

#### With Cache Components

We have two approaches to doing revalidation with Cache Components. They both work by revalidating a cache tag to let Next.js know that all pages depending on that cache tag are `STALE` (thus triggering an SWR build of the page on the next visit). Each strategy has their downsides. Both strategies suffer from minor changes in the page composition silently introducing revalidation bugs. However, both cache components strategies are far superior in terms of performance, to their non-cache components alternatives!

**FYI:** For both strategies you will need to introduce the concept of "topic" related tags, e.g `PAGES_TAG` so something like `sitemap.ts` gets revalidated!

##### Eager Tagging (Recommended)

Our goal is to over-query a little bit by requesting `sys.id` in our queries wherever we can. We then take the payload, extract those ids and add them as cache tags (alongside the original id or slug).

Here is an example:

```ts
export const getSecondaryHero = async (id: string, preview?: boolean) => {
  "use cache: remote";
  cacheLife("forever"); // custom tag

  const { data } = await getGraphQLClient(preview).query(SecondaryHeroQuery, {
    id,
    preview,
  });

  cacheTag(id, ...extractIds(data));

  return data;
};
```

This makes revalidation trivial. We set up our Contentful Webhook to call our API route and include the `id` of the changed entry, then call [revalidateTag()](https://nextjs.org/docs/app/api-reference/functions/revalidateTag) on that `id`.

Here is the payload for the simplest possible webhook:

```json
{
  "tag": "{ /payload/sys/id }"
}
```

Example of the API endpoint:

```ts
export async function POST(request: NextRequest) {
  const data = await request.json();
  const { id, slug } = z
    .object({
      id: z.string(),
      slug: z
        .object({
          "en-US": z.string(),
        })
        .optional(),
    })
    .parse(data);

  revalidateTag(id, "max");

  const actualSlug = slug?.["en-US"];

  if (actualSlug) {
    revalidateTag(actualSlug, "max");
    revalidateTag(PAGES_TAG, "max");
  }

  return NextResponse.json({}, { status: 200 });
}
```

###### Benefits

1. Dead simple logic
2. Does (almost) no API requests
3. Enforces little architectural restrictions

###### Downsides

1. Can have issues with publishing nested elements; can be mitigated by looking up & revalidating the parents of the `id`
2. Next.js hard limits a (single) cache entry to 128 tags (however you can have multiple cache entries associated with a page)
3. Edge-cases like the `slug` of a page need to be considered!

**Summary:** Eager tagging is simple to implement and easy to adapt. It is incredibly performant, as it relies on internal Next.js systems for the heavy lifting (thus not needing to do costly API calls). However, it runs into scaling issues for large cache entries which can be difficult to deal with.

##### Walk-Up (Legacy)

Given any id, all we have to do is _walk up_ the (Contentful) relations tree until we hit a "termination point". With Cache Components, a termination point can be any Cache Component, as invalidating a single tag is enough to cause any page associated with it to become stale.

Similar to the Eager Tagging approach, we set up a Webhook that calls our API route with the `id` that was updated. In our API route we now do the following API call in a loop to walk up the tree until we hit a termination point:

```ts
const result = await client.withAllLocales.getEntries({
  links_to_entry: id,
  include: 1,
});
```

The actual logic depends on your termination points.

**Note:** Make sure to implement failsafes like a max `depth` limit and a `visited` table

```ts
const onEntry = async (
  entry: Entry<EntrySkeletonType, "WITH_ALL_LOCALES", string>,
  visited: Set<string>,
  depth: number,
): Promise<Set<string>> => {
  // actual code
};
```

###### Benefits

1. Works regardless of Next.js tags
2. Easy to derive a `getIdsById` function (which can be handy)

###### Downsides

1. Easy to introduce errors
2. Does a lot of API calls (and as such can fail because of rate limits)
3. Errors can cause a massive number of API calls

**Summary:** Walk-Up is a reliable strategy that can be adapted to fit any content-model and doesn't run into scaling issues. However, it is both error prone and difficult to update. Termination points cannot easily be encoded into code, which makes it challenging to fully understand the system.

#### Without Cache Components

You can copy the Walk-Up implementation from Cache Components, but you will need to add a step to convert the ids to URLs that you can revalidate via [revalidatePath()](https://nextjs.org/docs/app/api-reference/functions/revalidatePath). This is easier than it sounds; you probably already have your page Content-Type (containing the `slug`) as a termination point.

## Contentful Apps

Contentful apps are great. They're easy to build and provide a lot of value.

### Forma36

Contentful allows you to use whatever UI library you want, however it does make sense to use the in-house [Forma36 library](https://f36.contentful.com/components) to continue the established look & feel of the Contentful UI.

The library offers some great components and is overall easy to use, however Contentful has historically been very slow to fix issues and to upgrade to newer versions (e.g. React 19). This can turn the library into a liability!

**Recommendation**: Use it, but consider alternatives if you're planning something ambitious (or check first).
