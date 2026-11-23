---
name: wordpress-elementor-publisher
description: "Publish or update posts on a self-hosted WordPress site built with Elementor, via the WordPress REST API. Use this skill whenever the user asks to update a post, publish new content, or change anything on their WordPress site — even if they just say \"update the blog post\" or \"publish this to the site\". Elementor stores content in its own field, and sites behind Cloudflare often strip the Authorization header, so standard API authentication and standard content updates do not work — this skill handles both.\n"
---

# WordPress Publisher for Elementor sites

## Site setup (fill this in before first use)

- **Site URL**: `https://YOUR-SITE.com`
- **Page builder**: Elementor (content lives in `_elementor_data`, NOT the standard WordPress content field)
- **Security layer**: Cloudflare (or similar) — it can silently strip `Authorization` headers, so Application Passwords may not work
- **Page cache**: `<your host or caching plugin, e.g. SiteGround, WP Rocket, LiteSpeed — or "none">`
- **SEO plugin**: `<e.g. Yoast — or "none">`
- **Authentication method**: Nonce from an active browser session (see Step 1)
- **Tools needed**: Claude in Chrome (to get the nonce and clear cache) + WordPress REST API

---

## Step 1: Get the authentication nonce

**Before anything else**, make sure the user has their WordPress admin open and logged in in Chrome.

Then retrieve the nonce from the browser session:

```javascript
// Run this in the WordPress admin tab via Claude in Chrome
wp.apiFetch({ path: '/wp/v2/posts' }).then(() => {}).catch(() => {});
// Or fetch it directly:
fetch('/wp-json/').then(r => r.headers.get('X-WP-Nonce'));
```

A simpler reliable approach: use `mcp__claude-in-chrome__javascript_tool` on the WordPress admin tab to run:

```javascript
window.wpApiSettings?.nonce || document.querySelector('#_wpnonce')?.value
```

If that doesn't return a nonce, navigate to any admin page (e.g. Posts list) and try again — a nonce is always present in an active admin session.

Use this nonce as the `X-WP-Nonce` header on all subsequent API calls instead of `Authorization`.

---

## Step 2: Find the post ID

If the user gives you a post title rather than an ID, find it:

```
GET https://YOUR-SITE.com/wp-json/wp/v2/posts?search=<post title>&_fields=id,title,status
Headers: X-WP-Nonce: <nonce>
         Cookie: <copy from browser session>
```

Confirm the correct post with the user before proceeding.

---

## Step 3: Fetch the existing Elementor data

**Always fetch before updating** — you need the full widget structure with all widget IDs to know where each piece of content goes.

```
GET https://YOUR-SITE.com/wp-json/wp/v2/posts/<post_id>?context=edit
Headers: X-WP-Nonce: <nonce>
         Cookie: <copy from browser session>
```

The `_elementor_data` field in the response is a JSON string containing an array of widgets. Each widget has:
- `id` — unique widget identifier (e.g. `"c27cd74"`)
- `widgetType` — e.g. `"text-editor"`, `"heading"`, `"accordion"`
- `settings.editor` or `settings.title` — where the content lives

Parse this structure carefully. Build a mapping of widget IDs to their purpose (intro, section heading, FAQ item, etc.) before writing any new content.

---

## Step 4: Map new content to widget IDs

Match each piece of new content to the correct widget ID from the structure you fetched. Keep all existing widgets intact — only change the `settings.editor` or `settings.title` values for the widgets you're updating. Do not add or remove widgets unless the user explicitly asks, or an approved draft has a section the page doesn't have yet.

**Adding a section:** look at how the page nests its widgets (often each widget sits in its own top-level container). Copy an existing heading block and text block (deep copy), give each container and widget a new random 7-character hex id, set the new title and text, and insert them at the right position. The new section then matches the site's styling. Never copy or move image widgets unless asked: keep the images where they are.

---

## Step 5: Send the update (chunking if content is large)

If the full updated `_elementor_data` JSON string is longer than ~7,000 characters, **split the transfer into 3 chunks** to avoid a 45-second browser timeout. Reassemble in browser memory before the final API call.

**Also send `content`:** build plain HTML of the whole article from the updated widgets, in order (headings as their header level, text-editor HTML as is, images as `<img>`), and send it as `content` in the same request. Elementor renders the page from `_elementor_data`, but feeds, search engines and AI crawlers read `content`; if it isn't updated they see the old article.

**Small content (< 7,000 chars):** send directly via the REST API:

```
POST https://YOUR-SITE.com/wp-json/wp/v2/posts/<post_id>
Headers: X-WP-Nonce: <nonce>
         Cookie: <copy from browser session>
         Content-Type: application/json

Body: {
  "meta": {
    "_elementor_data": "<updated JSON string>"
  }
}
```

**Status:** for a new post, add `"status": "draft"` unless the user explicitly says to publish. For a post that is already live (a refresh), send no `status` at all, so it stays published: switching it to draft takes the article offline.

**Large content (> 7,000 chars):** use Claude in Chrome to run JavaScript that:
1. Receives the content in 3 chunks via separate `javascript_tool` calls
2. Reassembles them: `window._chunk1 + window._chunk2 + window._chunk3`
3. Makes the `fetch()` call from inside the browser using the existing session cookies

---

## Step 5b: Title, excerpt and SEO fields

- The post **title** is the H1: send it as `title` when it changes.
- The **excerpt** often shows as a subtitle or summary, depending on the theme. When the article changes, check whether the excerpt still fits; propose new text and send it as `excerpt` once the user confirms.
- If the site uses Yoast, its fields are post meta: `_yoast_wpseo_title`, `_yoast_wpseo_metadesc`, `_yoast_wpseo_focuskw`.

---

## Step 6: Verify the update

Fetch the post back and confirm the new content is in `_elementor_data`:

```
GET https://YOUR-SITE.com/wp-json/wp/v2/posts/<post_id>?context=edit
```

Check that the widget content matches what was sent. If it doesn't match, something went wrong with the update — do not proceed to cache clearing.

---

## Step 7: Clear the Elementor cache

Even after a successful update, the page will show old content until the cache is cleared. This is **always required**.

Use Claude in Chrome to:
1. Navigate to `https://YOUR-SITE.com/wp-admin/admin.php?page=elementor-tools`
2. Click the "Clear Files & Data" button
3. Wait for the success confirmation
4. Then purge the page cache, if the site has one (e.g. **Purge SG Cache** in the admin bar on SiteGround, or your caching plugin's purge button). Without it, visitors who aren't logged in can keep seeing the old page.
5. Check the public page as a logged-out visitor: `fetch(url, {credentials:'omit'})` from a site tab, and look for a sentence from the new text.

---

## Step 8: Confirm

Ask the user to open the post preview and confirm the new content is showing. If it's still showing old content after the cache clear, ask them to do a hard refresh (Cmd+Shift+R on Mac, Ctrl+Shift+R on Windows).

---

## Common errors and fixes

| Error | Likely cause | Fix |
|-------|-------------|-----|
| `401 Unauthorized` | Cloudflare stripped the Authorization header | Switch to nonce auth (Step 1) |
| Update succeeds but old content shows | Elementor or page cache | Clear both caches (Step 7) |
| Update succeeds but only standard content changed | Updated `post_content` instead of `_elementor_data` | Re-read Step 3–5 |
| Browser timeout during content transfer | Content too large for single JS call | Split into 3 chunks (Step 5) |
| Nonce not found | Admin not logged in, or wrong tab | Ask the user to open WordPress admin in Chrome |
| A tab opens on the WordPress login screen | The user's session has expired | Ask them to log in again in Chrome; never enter the password for them. Reading published posts works without login (`/wp-json/wp/v2/posts` is public) |

---

## What to tell the user

For a new post: save as **draft** unless they explicitly say to publish, and after the preview looks right, tell them it's ready to publish from their dashboard.

For a refresh of a live post: it goes live as soon as the update is saved, so do it only after their explicit approval in the conversation, then tell them it's live and what you checked on the public page.
