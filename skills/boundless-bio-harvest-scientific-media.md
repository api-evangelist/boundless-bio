---
name: Harvest Boundless Bio scientific posters and media
description: Enumerate the 255-item boundlessbio.com media library over the public WordPress REST API to find the company's scientific PDFs — conference posters and publication assets — and pick the right size variant for imagery.
api: openapi/boundless-bio-content-openapi.yml
operations: [getMedia, getMediaById, getPages]
method: generated
generated: '2026-08-08'
---

# Harvest Boundless Bio scientific posters and media

`/wp/v2/media` exposes 255 attachments. Public, no credentials. On this deployment it is more than a
image library: **the company's primary scientific material is stored here as PDFs** — conference
posters and publication assets that are otherwise only reachable through links on the Publications
page.

## Step 1 — find the scientific documents (`getMedia`)

Filter to non-image attachments first — that is where the value is:

```
GET /wp/v2/media?per_page=100&mime_type=application/pdf&orderby=date&order=desc&_fields=id,slug,date,modified,title,source_url,mime_type,media_type,post,alt_text,caption,description
```

Observed most-recent item on 2026-08-08:

```json
{
  "id": 2213,
  "slug": "aacr2026_bbiposter_final",
  "media_type": "file",
  "mime_type": "application/pdf",
  "post": null,
  "source_url": "https://boundlessbio.com/wp-content/uploads/2026/04/AACR2026_BBIPoster_Final.pdf"
}
```

Note `media_type` is **`file`** for PDFs, not `image` — the `media_type` enum is `image`, `video`,
`text`, `application`, `audio`, `file`. Filtering on `mime_type=application/pdf` is the precise
route.

`post: null` means the PDF is a **library-only upload with no parent page**. There is no
`featured_media` or `parent` pointer to walk back from, so enumeration is the only way to find it —
which is exactly why this skill exists.

## Step 2 — enumerate everything else

```
GET /wp/v2/media?per_page=100&orderby=date&order=desc&_fields=id,slug,date,modified,title,source_url,mime_type,media_type,filesize,post,alt_text
```

- `per_page` max is **100**; 255 items means 3 pages. Follow `Link: rel="next"` and stop at
  `X-WP-TotalPages`.
- `_fields` is not optional at this size — the default representation inlines the whole
  `media_details.sizes` map for every item.
- `&parent=<page id>` narrows to attachments uploaded to a specific page.
- `&modified_after=<last run>` is the correct incremental-sync filter.
- `&search=<term>` matches title, caption and description.

## Step 3 — pick the right image variant (`getMediaById`)

```
GET /wp/v2/media/{id}
```

`media_details.sizes` is a map of every generated variant (`thumbnail`, `medium`, `large`, `full`),
each with its own `source_url`, `width`, `height`, `mime_type` and `filesize`. **Take the smallest
variant that satisfies your requirement.** The top-level `source_url` is the original upload and is
frequently many times larger than anything you need.

`alt_text`, `caption.rendered` and `description.rendered` carry the descriptive text — where the
useful context lives for an asset you cannot see. Many library items have **empty `alt_text`**
(including the AACR poster above); do not assume it is populated, and do not infer content from the
filename alone.

## Step 4 — tie an asset to its page (`getPages`)

`post` is the id of the page the attachment was uploaded to, or `null` for library-only uploads.
Resolve with `GET /wp/v2/pages/{id}`. Going the other way, page imagery is already **inlined as full
attachment objects inside `acf.modules`**, so for page graphics you do not need this endpoint at all
— see `boundless-bio-read-corporate-pages.md`.

The [Publications page](https://boundlessbio.com/publications/) is the human-facing index of the
company's scientific output; its `acf.modules` are the structured version of the same list.

## Etiquette and errors

- robots.txt asks for `Crawl-delay: 10`. Three index pages is cheap; downloading 255 originals is
  not. Fetch only what you need and pace the binary downloads.
- `cache-control: max-age=600` behind Cloudflare and WP Engine — re-polling faster gains nothing.
- Errors are the WordPress envelope `{code, message, data:{status}}`. `rest_post_invalid_id` (404)
  for a bad attachment id; `rest_invalid_param` (400) for `per_page` over 100 or an unregistered
  `media_type` or `mime_type` value.
- Attachments are copyrighted company material. Enumerating the metadata is a public read;
  redistributing the PDFs is a separate question this skill does not answer.
