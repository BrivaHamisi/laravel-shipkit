---
name: seo-crawlability
description: "Serve dynamic robots.txt, sitemap.xml and llms.txt from Laravel with absolute URLs from APP_URL, keep staging sites out of Google, and mark pages noindex with a meta tag plus X-Robots-Tag header. Use when adding SEO basics, a sitemap, robots.txt, llms.txt for AI assistants, fixing pages indexed from staging, or preparing a Laravel site for Google Search Console."
license: MIT
metadata:
  author: BrivaHamisi
---

# SEO crawlability: robots.txt, sitemap.xml, llms.txt

Three routes that tell crawlers what to index, plus a middleware for what not to index.

- **`/robots.txt`**: in production, disallows private paths and links the sitemap. **Everywhere else it disallows everything**, so a staging or preview copy can never compete with the real site in search results.
- **`/sitemap.xml`**: the home page, named routes from config, and any content you register. Absolute URLs from `APP_URL`, with `lastmod` dates.
- **`/llms.txt`**: a plain-text summary for AI assistants, following the llmstxt.org format.
- **`NoIndex` middleware**: `noindex, follow` as both a meta-tag variable (`$robots`) and an `X-Robots-Tag` header.

## Install

1. Copy the files in `stubs/` to the same paths, dropping `.stub`: `config/seo.php`, `app/Support/Seo/Sitemap.php`, `app/Http/Controllers/SeoController.php`, `app/Http/Middleware/NoIndex.php`, `routes/seo.php`, `resources/views/seo/*.blade.php`, `tests/Feature/SeoCrawlabilityTest.php`.
2. Add `require __DIR__.'/seo.php';` to `routes/web.php`.
3. **Delete `public/robots.txt`.** The web server serves static files before Laravel, so the dynamic route never runs while that file exists. This is the most common reason "my robots.txt changes do nothing".
4. Set `APP_URL` correctly in production, including `https://`. Every URL in the sitemap comes from it.
5. List public pages in `config/seo.php` → `sitemap_routes`, and register database content in `AppServiceProvider::boot()`:
   ```php
   use App\Support\Seo\Sitemap;

   app(Sitemap::class)->add(fn () => Post::query()->whereNotNull('published_at')->get()
       ->map(fn (Post $post) => ['loc' => route('posts.show', $post), 'lastmod' => $post->updated_at]));
   ```
   Sources are closures, so nothing is queried until `/sitemap.xml` is requested.
6. Fill in `seo.llms` (summary, key facts, pages) so AI assistants describe the site correctly.
7. In your layout's `<head>`:
   ```blade
   <meta name="robots" content="{{ $robots ?? 'index, follow, max-image-preview:large' }}">
   ```
   Then put `->middleware(NoIndex::class)` on pages that shouldn't be in search results: thank-you pages, internal search results, demo or template pages.
8. Submit `https://your-site/sitemap.xml` in Google Search Console and Bing Webmaster Tools.

## Rules

- **robots.txt is advice, not security.** It asks polite crawlers not to crawl, and it publishes the paths it lists. Private areas still need auth.
- **`Disallow` doesn't remove a page from Google.** A disallowed page can still be indexed from links, and Google can't see a `noindex` on a page it isn't allowed to crawl. To remove a page, allow crawling and send `noindex`.
- **Only list canonical, indexable, 200-status URLs in the sitemap:** no redirects, drafts or noindex pages. Keep it under 50,000 URLs per file.
- **Blade and XML:** a literal `<?xml` in a Blade file is parsed as PHP, so the view builds it from pieces.
- **llms.txt is text/plain.** Print values with `{!! !!}`. With `{{ }}`, apostrophes come out as `&#039;`.
- **Sitemap is a singleton** (`#[Singleton]`, Laravel 12+). On Laravel 11, register it with `$this->app->singleton(Sitemap::class)`, or your sources vanish.
- Pair with `json-ld-structured-data` (rich results) and `social-share-previews` (link previews). Together they cover what search engines and social apps read.
