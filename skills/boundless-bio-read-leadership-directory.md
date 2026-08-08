---
name: Read the Boundless Bio leadership directory
description: Enumerate the 27 published Boundless Bio Leadership records — executive team, board of directors and scientific / clinical advisors — with biographies, job titles, headshots and LinkedIn URLs, over the public WordPress REST API.
api: openapi/boundless-bio-content-openapi.yml
operations: [getWhoWeAre, getWhoWeAreById, getCategories, getMediaById]
method: generated
generated: '2026-08-08'
---

# Read the Boundless Bio leadership directory

`/wp/v2/who-we-are` is the `who-we-are` custom post type, registered with the label **Leadership**.
27 published records. Public, no credentials. This is the richest collection on the surface — it is
the only one where `content.rendered` is actually populated.

## Step 1 — enumerate (`getWhoWeAre`)

```
GET /wp/v2/who-we-are?per_page=100&orderby=title&order=asc&_fields=id,slug,title,excerpt,link,categories,featured_media,acf
```

- `per_page` max is **100**; 27 records fit in one page. `X-WP-Total: 27` and
  `X-WP-TotalPages` confirm the size — read them rather than assuming.
- `_fields` matters here: the default representation inlines a large `yoast_head` HTML blob plus
  `yoast_head_json` on every record.
- Add `content` to `_fields` when you want the full biography (about 1,300 characters per record).
  `excerpt.rendered` is the short form.

What you get per record:

- `title.rendered` — the person's name, as published, including post-nominals
  (e.g. `Dr. Robert Doebele, M.D., Ph.D.`).
- `content.rendered` / `excerpt.rendered` — the biography as HTML. Strip tags before use.
- `acf.title` — **the job title**, and it is HTML-wrapped: `"<p>Chief Medical Officer</p>\n"`.
  Strip the `<p>` wrapper and trim.
- `acf.linkedin` — a link object `{title, url, target}`. Populated on **all 27** records.
- `acf.twitter` — a link object or an empty string. Usually empty.
- `acf.under_bio_text` — optional supplementary text.
- `featured_media` — the headshot attachment id. `better_featured_image` is also inlined by the
  Better REST API Featured Images plugin, so you usually do not need a second request.

## Step 2 — do NOT segment by category (`getCategories`)

```
GET /wp/v2/categories?per_page=20&_fields=id,name,slug,count
```

Four terms are registered:

| id | name | slug | count |
|---|---|---|---|
| 2 | Board of Directors | `board-of-directors` | 4 |
| 7 | Leadership | `leadership` | 4 |
| 4 | Scientific / Clinical Advisors | `scientific-founders` | 5 |
| 1 | Uncategorized | `uncategorized` | 0 |

**Two traps, and they will both bite you.**

1. The three populated terms account for **13 of 27** records. The other **14 return
   `categories: []`**. `GET /wp/v2/who-we-are?categories=2,7,4` therefore silently returns about
   half the team. If you need everyone, enumerate the whole collection and derive the role grouping
   from `acf.title` instead.
2. The advisors term's slug (`scientific-founders`) does not match its display name
   (`Scientific / Clinical Advisors`). Filter on the **id** or the **slug**, never on the name.

## Step 3 — a single person (`getWhoWeAreById`)

```
GET /wp/v2/who-we-are/{id}
```

Or go straight to the public page via `link` — `https://boundlessbio.com/who-we-are/{slug}/`.

## Step 4 — the headshot (`getMediaById`)

```
GET /wp/v2/media/{featured_media}
```

`media_details.sizes` is a map of generated variants (`thumbnail`, `medium`, `large`, `full`), each
with its own `source_url`, `width`, `height` and `filesize`. **Take the smallest variant that meets
your requirement** — the top-level `source_url` is the original upload. Prefer the inlined
`better_featured_image` on the record to avoid the request entirely.

## Personal data — read this before you store anything

These records are **published executive biographies**: the same content the public
`/who-we-are/` page renders to any visitor, put there deliberately by the company. Treating them as
public is appropriate.

That is **not** true of `/wp/v2/users`. On this deployment that route answers **anonymously with
HTTP 200** (most WordPress sites return 401) and discloses the display names and author-archive
slugs of internal CMS staff accounts. **Do not enumerate it, do not join it to these records, and do
not store its output.** It is documented as an exposure in
`data-model/boundless-bio-data-model.yml` and is deliberately excluded from every skill in this
repository.

## Errors and etiquette

- Errors are the WordPress envelope `{code, message, data:{status}}` — not RFC 9457.
  `rest_post_invalid_id` (404) for a bad id; `rest_invalid_param` (400) for `per_page` over 100 or a
  non-enum `orderby`; `rest_post_invalid_page_number` (400) for paging past the end.
- robots.txt asks for `Crawl-delay: 10`. One request for the whole collection is well within that —
  there is no reason to page this endpoint at all.
- Responses carry `cache-control: max-age=600` behind Cloudflare; re-polling faster than 10 minutes
  gains you nothing.
- Use `modified_after=<last run ISO8601>` for incremental sync.
