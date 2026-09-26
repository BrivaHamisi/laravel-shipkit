---
name: json-ld-structured-data
description: "Add schema.org JSON-LD structured data to Laravel Blade pages as one request-scoped @graph: Organization/NGO/LocalBusiness, WebSite with sitelinks search, BlogPosting, BreadcrumbList and FAQPage, safely escaped. Use when adding structured data, schema markup, rich results, Google knowledge panel details, breadcrumbs or FAQ snippets, or fixing Rich Results Test errors."
license: MIT
metadata:
  author: BrivaHamisi
---

# JSON-LD structured data

Search engines and AI assistants read JSON-LD to learn who publishes a site, what each page is, and how pages relate. This skill prints **one `@graph` per page**. The organisation and website appear on every page, and each view adds its own entities: an article, breadcrumbs or FAQs. Everything is linked by `@id`.

## Install

1. Copy from `stubs/`, dropping `.stub`: `app/Support/StructuredData.php`, `app/Providers/StructuredDataServiceProvider.php`, `config/structured-data.php`, `tests/Feature/StructuredDataTest.php`.
2. Register the provider in `bootstrap/providers.php` (Laravel 11+).
3. Fill in `config/structured-data.php`: the most specific type (`NGO`, `LocalBusiness`, `Restaurant`...), logo, address and official profiles (`same_as`).
4. In the layout's `<head>`:
   ```blade
   {{ app(\App\Support\StructuredData::class)->toScript() }}
   ```
5. In page views, before the layout renders:
   ```blade
   @php
       $ld = app(\App\Support\StructuredData::class);
       $ld->addArticle([
           'url' => route('posts.show', $post),
           'headline' => $post->title,
           'description' => $post->excerpt,
           'image' => $post->image_url,
           'published_at' => $post->published_at,
           'updated_at' => $post->updated_at,
           'author' => $post->author_name,
       ]);
       $ld->addBreadcrumbs([['name' => 'Home', 'url' => url('/')], ['name' => 'Blog', 'url' => route('posts.index')], ['name' => $post->title]]);
   @endphp
   ```
   Also `addFaqs([['question' => ..., 'answer' => ...]])`, or `add([...])` for any other schema.org type (Event, Product, Course...).
6. Validate a few URLs with Google's Rich Results Test and the Schema.org validator.

## Rules

- **Request-scoped, and reset after each response.** A plain singleton makes entities from one page appear on the next in tests and on Octane. The provider registers it as `scoped` and forgets it on `RequestHandled`. The test "entities from one request never leak into the next" guards this.
- **Escape for HTML, not only JSON.** `json_encode` with `JSON_HEX_TAG | JSON_HEX_AMP` turns `<` and `>` into `<`/`>`, so a post titled `</script><script>...` can't inject a script. Never build JSON-LD by concatenating strings in Blade.
- **Mark up only what the page shows.** FAQ answers, prices and ratings in JSON-LD must be visible on the page, or Google may ignore the page's markup or apply a manual action.
- **Absolute URLs everywhere** (`url()`, `route()`, `asset()`), and an image for articles: without one they aren't eligible for article rich results.
- **Blade section order:** with `@extends`, the child view renders before the layout, so entities added in sections are ready when `<head>` prints. With component layouts (`<x-layout>`), add entities before the component tag, or in the controller.
- **One organisation `@id`** (`url('/').'#organization'`) referenced as `publisher` and `author` fallback. Don't repeat the organisation's details on every entity.
- Blade gotcha: an inline `@php(...)` before a block `@php ... @endphp` in the same file breaks parsing. Use the block form.
