# WordPress Publisher for Elementor sites

A Claude skill that publishes and updates posts on a self-hosted WordPress site built with Elementor.

## The problem it solves

Two things break the usual approach.

1. **Elementor keeps your content in its own field.** WordPress has a standard content field, and that's what most API tutorials update. Elementor ignores it and renders the page from `_elementor_data`. So the update "succeeds", and your page still shows the old article.
2. **Cloudflare can strip the login header.** The usual way to log in through the API is an Application Password sent in the `Authorization` header. Behind Cloudflare, that header can disappear on the way, and every request comes back `401 Unauthorized`.

The skill works around both. It uses your logged-in browser session instead of a password, updates the Elementor data widget by widget, and then clears the caches so the new version actually shows up.

## What it does

- Finds a post by title and confirms it with you before touching anything
- Reads the page's Elementor structure and maps your new text to the right widgets
- Leaves your layout, styling and images alone
- Adds new sections that match the existing styling, when your draft needs one
- Updates the plain-HTML version too, so feeds, search engines and AI crawlers see the new article
- Saves new posts as drafts unless you say otherwise
- Updates title, excerpt and Yoast SEO fields when they need it
- Clears the Elementor cache and your page cache, then checks the public page as a logged-out visitor

It never enters your password. If your session has expired, it asks you to log in.

## What you need

- A self-hosted WordPress site (not WordPress.com) with Elementor
- Claude with Claude in Chrome, since the skill works through your browser session
- Your WordPress admin open and logged in, in Chrome

## Install

**In the Claude app:** download this folder as a zip (green **Code** button → **Download ZIP**), then upload it under **Settings → Capabilities → Skills**.

**In Claude Code:** copy the folder into `~/.claude/skills/`.

## Set it up for your site

Open `SKILL.md` and fill in the **Site setup** block at the top:

- Replace `YOUR-SITE.com` with your domain (it appears in several steps, so use find and replace)
- Name your page cache, if you have one (SiteGround, WP Rocket, LiteSpeed...)
- Name your SEO plugin, if you use one

That's it. Then ask Claude something like "update the post about X with this draft" and it takes it from there.

## A note on how it got here

This skill is the result of a lot of detours. Every rule in it is there because something broke on a real post first: the page that didn't update, the cache that kept the old version, the refresh that accidentally took a live article offline. If you hit a case it doesn't handle, open an issue and tell me what happened.

## About me

I'm Ina Toncheva. I write about building real AI systems for content marketing, the ones that work in practice, with the messy experiments included. If that's useful to you, my newsletter is [The Irreplaceable Marketer](https://inatoncheva.substack.com/).

## License

MIT. Use it, adapt it, share it. Keep the credit.
