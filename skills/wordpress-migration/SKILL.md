---
name: wordpress-migration
description: "Migrate content from a WordPress site to Laravel using only its SQL dump: stream-read phpMyAdmin or mysqldump files, export published posts and pages with clean HTML (no Gutenberg comments or shortcodes), categories, tags, authors, featured images and correct UTC dates, and filter bot/spam entries from imported form data. Use when moving from WordPress to Laravel, importing a .sql backup, seeding blog posts from WordPress, or cleaning spam out of imported donations, sign-ups or contact entries."
license: MIT
metadata:
  author: BrivaHamisi
---

# WordPress to Laravel, from the SQL dump

You rarely get a running WordPress site to migrate from. More often you get a `.sql` backup. This skill reads it directly, one row at a time: there's no MySQL import, no WordPress install and no plugins. It turns the content into clean JSON for a normal Laravel seeder.

## Install

Copy from `stubs/`, dropping `.stub`:

- `app/Support/WordPress/WordPressDump.php`: streaming reader for phpMyAdmin and mysqldump files. It handles multi-line strings, escapes, commas and quotes, and never loads the whole file.
- `app/Support/WordPress/WordPressContent.php`: turns post content into plain HTML.
- `app/Support/SpamSignals.php`: bot and attack heuristics for imported form data.
- `app/Console/Commands/ExportWordPressPosts.php`: `php artisan wordpress:export-posts`.
- `tests/Feature/WordPressMigrationTest.php`, plus `tests/Fixtures/wordpress-dump.sql` and `tests/Fixtures/wordpress-mysqldump.sql`.

## Migrate posts

```bash
php artisan wordpress:export-posts ~/Downloads/site.sql --timezone=Africa/Nairobi
# storage/app/wordpress/posts.json
php artisan wordpress:export-posts ~/Downloads/site.sql --type=page --output=storage/app/wordpress/pages.json
```

Each entry has `wordpress_id`, `title`, `slug`, `excerpt`, `html`, `published_at` (UTC, ISO 8601), `author`, `categories`, `tags` and `featured_image` (the old URL). Then seed idempotently:

```php
foreach (json_decode(File::get(storage_path('app/wordpress/posts.json')), true) as $post) {
    Post::updateOrCreate(['slug' => $post['slug']], [
        'title' => $post['title'],
        'body' => $post['html'],
        'excerpt' => $post['excerpt'] ?? Str::limit(strip_tags($post['html']), 160),
        'published_at' => $post['published_at'],
        'category_id' => Category::firstOrCreate(['name' => $post['categories'][0] ?? 'News'])->id,
    ]);
}
```

For featured images, download each `featured_image` once (or copy it from the `wp-content/uploads` backup) and store it with the `image-uploads` skill. Skip the upload when the post already has one, so re-seeding doesn't duplicate files.

## WordPress data gotchas this handles

- **mysqldump omits column names** from `INSERT` statements. They're read from `CREATE TABLE`. If a dump has neither, re-export with phpMyAdmin or `mysqldump --complete-insert`.
- **Dates:** `post_date` is in the old site's timezone and `post_date_gmt` is UTC, but it's `0000-00-00` for some posts. The exporter uses the GMT date, and falls back to `--timezone`.
- **Classic-editor posts have no `<p>` tags.** WordPress adds them only when displaying (`wpautop`). The cleaner adds paragraphs and line breaks.
- **Gutenberg** wraps blocks in `<!-- wp:... -->` comments, which are removed. **Shortcodes** (`[contact-form-7]`, `[gallery]`) are removed. `[caption]` keeps its image and text. Search the output for leftover `[` if the site used unusual shortcodes.
- **Titles and term names are HTML-escaped** (`News &amp; Updates`) and are decoded. "Uncategorized" is dropped.
- **Featured images** are `_thumbnail_id` meta pointing at an attachment post, whose `guid` is the file URL.
- **Table prefix:** it's `wp_` by default, but many hosts use something else. Check the dump and pass `--prefix=`.
- **Internal links** in content still point at the old domain. Replace `https://old-site/...` with new routes after importing. Keep old slugs to preserve search rankings, or add 301 redirects.

## Importing form data (donations, sign-ups, messages)

Old WordPress forms collect years of bot traffic: serialized PHP objects injected into name fields, `@example.com` and `@mailinator.com` addresses, card-testing floods from `name+1@`, `name+2@`..., and random-letter names. Before importing unverified entries:

```php
$floods = SpamSignals::plusAddressCounts($entries->pluck('email'));

$entries->reject(fn ($entry) => $entry['status'] !== 'completed'
    && SpamSignals::looksFake($entry['first_name'], $entry['last_name'], $entry['email'], $floods));
```

Never drop **completed payments** on heuristics: money arrived, so it's real. Make the import idempotent (`updateOrCreate` on the WordPress ID), and delete entries that an earlier run imported and that now look fake. Keep personal data from the dump **out of git**: import from a local file path, never commit the `.sql`.
