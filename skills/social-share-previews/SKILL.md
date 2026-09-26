---
name: social-share-previews
description: "Make Laravel pages show a proper title, description and image when shared on WhatsApp, Facebook, LinkedIn, X, Slack and iMessage, using one Blade component for Open Graph, Twitter card, canonical and meta description tags. Use when link previews show no image or the wrong text, adding og:image or Open Graph tags, or setting page titles and meta descriptions."
license: MIT
metadata:
  author: BrivaHamisi
---

# Social share previews (Open Graph and Twitter cards)

When someone pastes your link into WhatsApp or LinkedIn, the app fetches the page and reads a few `<meta>` tags. Get them wrong and the preview is a bare URL. `<x-share-meta />` prints them all correctly, with site-wide fallbacks.

## Install

1. Copy from `stubs/`, dropping `.stub`: `resources/views/components/share-meta.blade.php`, `config/share.php`, `tests/Feature/ShareMetaTest.php`.
2. Create a default share image: **1200 × 630, JPEG, under 300 KB**, at `public/images/share/default.jpg`, or change `config('share.image')`. Put the logo and name in the middle. Some apps crop the edges into a square.
3. In the layout `<head>`, replace the existing `<title>` and description tags:
   ```blade
   <x-share-meta :title="$title ?? null" :description="$description ?? null" :image="$shareImage ?? null" :type="$ogType ?? 'website'" :robots="$robots ?? null" />
   ```
4. Pass details from each page: a blog post sets its title, excerpt, a **JPEG share-size image URL** and `type="article"`.
5. Check real previews with the Facebook Sharing Debugger, LinkedIn Post Inspector, and by sending the link to yourself on WhatsApp.

## Why previews break (and what this component does about it)

- **WebP images.** WhatsApp and several other apps often show no image for WebP `og:image`. Serve a JPEG or PNG share size. The `image-uploads` skill generates a 1200 × 630 `share.jpg` for every upload.
- **Relative URLs.** `og:image` must be absolute, with `https://` in production. Everything here goes through `url()`/`asset()`, so set `APP_URL` correctly. A test asserts that every URL is absolute.
- **Wrong size.** 1200 × 630 (1.91:1) works everywhere. Images under 200 × 200 are ignored. The default image also declares its width and height, so the first share renders immediately.
- **HTML in descriptions.** The description is stripped of tags, squished and cut to 160 characters.
- **Caching.** WhatsApp and Facebook cache previews for days. After fixing tags, re-scrape in the Facebook Sharing Debugger. For WhatsApp, test with a new URL (for example with `?v=2`).
- **Pages behind login or bot protection** can't be previewed: the crawler gets the login page. Keep shareable pages public, and don't block `facebookexternalhit`, `WhatsApp`, `LinkedInBot` or `Twitterbot`.
- **One canonical URL.** `og:url` and `<link rel="canonical">` use the same URL, so shares of `?utm=` variants count towards one page.
