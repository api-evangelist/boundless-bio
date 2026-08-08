---
name: Read Boundless Bio corporate pages (the acf.modules trap)
description: Read the 9 published boundlessbio.com pages over the public WordPress REST API, working around the fact that content.rendered is empty on every page and the real copy is served in the ACF flexible-content modules array.
api: openapi/boundless-bio-content-openapi.yml
operations: [getPages, getPagesById, getTypes, getSearch]
method: generated
generated: '2026-08-08'
---

# Read Boundless Bio corporate pages

9 published pages. Public, no credentials. **This surface has one trap that makes a naive read
return nothing, so start there.**

## The trap: `content.rendered` is empty on every page

Verified on 2026-08-08 against `GET /wp/v2/pages/255` (Contact Us) and `GET /wp/v2/pages/305`
(Homepage):

```json
{ "id": 255, "slug": "contact", "content": { "rendered": "" }, "excerpt": { "rendered": "" } }
```

The copy is not missing. The site is built with **Advanced Custom Fields flexible content**, and the
entire page body is served in the public **`acf.modules`** array — one entry per layout block, each
tagged with `acf_fc_layout`. An agent that reads `content.rendered` will conclude boundlessbio.com is
an empty site.

The `who-we-are` (Leadership) type is the **exception** — its `content.rendered` is populated
normally. See `boundless-bio-read-leadership-directory.md`.

## Step 1 — enumerate (`getPages`)

```
GET /wp/v2/pages?per_page=100&_fields=id,slug,title,link,parent,menu_order,modified
```

`X-WP-Total: 9`. The page tree is flat — `parent` is 0 on every page. Published slugs at harvest:

| slug | title |
|---|---|
| `homepage` | Homepage |
| `why-we-are-here` | Why We're Here |
| `what-we-do` | What We Do |
| `who-we-are` | Who We Are |
| `publications` | Publications |
| `work-with-us` | Work With Us |
| `contact` | Contact Us |
| `terms-of-use` | Terms Of Use |
| `privacy-policy` | Privacy Policy |

## Step 2 — read the body (`getPagesById`)

```
GET /wp/v2/pages/{id}
```

Do **not** pass `_fields=content`. Request `acf` (or take the full object) and walk
`acf.modules[]`. Each entry carries `acf_fc_layout` naming the block type, then that block's own
fields. Observed on page 255:

| `acf_fc_layout` | carries |
|---|---|
| `hero` | `hero_static_bg_image` (resolved attachment object), `hero_static_tagline`, `hero_static_headline`, `hero_static_text`, `hero_static_cta` |
| `what_is_ecdna` | `what_is_ecdna_bg`, `what_is_ecdna_items`, `what_is_ecdna_image` |
| `location` | `location_image`, `location_press`, `location_address` |
| `page_callout` | `page_callout_bg_image`, `page_callout_headline`, `page_callout_cta` |

**Field names are per-layout and are not a stable contract.** They come from the theme's ACF field
groups, not from WordPress core, and they will change if the theme changes. Discover them by walking
the object; never hard-code a path without a fallback.

Two conveniences worth knowing:

- ACF **resolves image fields to full attachment objects inline** — `ID`, `url`, `width`, `height`,
  `filesize`, `mime_type`, `uploaded_to`, `alt`, `caption`. Page imagery needs no follow-up request
  to `/wp/v2/media`.
- Text fields come back as HTML. Strip tags before use.

If you only need prose and not structure, fetching the rendered HTML page at `link` is the simpler
path — but send a browser-class `User-Agent` and honour the crawl delay.

## Step 3 — confirm the type surface (`getTypes`)

```
GET /wp/v2/types
```

12 registered types. Beyond WordPress core the only custom type is `who-we-are` (rest_base
`who-we-are`, label "Leadership"). Use this to confirm nothing new has been registered before
assuming the page set is the whole story.

## Step 4 — find a page without knowing its slug (`getSearch`)

```
GET /wp/v2/search?search=<term>&per_page=10&subtype=any
```

Returns lightweight pointers `{id, title, url, type, subtype}`. `type` is the generic bucket
(`post`); **`subtype` is the real content type**. `_links.self.href` already gives the resolved REST
URL, so no `/wp/v2/types` lookup is needed to dereference a hit.

Do not confuse this with `/wp/v2/global/search`, a theme AJAX route in a different namespace that
returns **HTTP 500** with `{"code":"front_end_ajax_search","message":"No results","data":null}` when
a query has no matches. It is not part of `wp/v2` and is not modelled in the OpenAPI.

## What is not here

No blog: `/wp/v2/posts` has `X-WP-Total: 0` and `/feed/` is a valid RSS channel with zero items.
Corporate news and SEC filings are published on the separate investor site at
`investors.boundlessbio.com`, which is third-party (gcs-web) infrastructure and is not part of this
API. Nothing about the clinical pipeline, trial data or ECHO/Spyglass internals is exposed beyond
the marketing copy in these pages.

The site footer's "What's New" link points at `https://boundlessbio.com/whats-new/`, which **404s** —
do not build a sync on it.

## Errors and etiquette

- WordPress envelope `{code, message, data:{status}}`. `rest_post_invalid_id` (404) for a bad page
  id; `rest_invalid_param` (400) for `per_page` over 100.
- `Crawl-delay: 10` in robots.txt; `cache-control: max-age=600` behind Cloudflare. Nine pages is one
  cheap pass — do not poll it.
- Use `modified_after=<last run ISO8601>` for incremental sync.
