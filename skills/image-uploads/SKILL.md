---
name: image-uploads
description: "Store Laravel image uploads as responsive WebP sizes plus JPEG share images for WhatsApp/Facebook previews, with face-friendly cropping of portrait photos, logo-safe 'contain' sizes, per-size URLs on models, safe deletion and a regenerate command. Use when handling image uploads, thumbnails, resizing, WebP conversion, Intervention Image, og:image sizes, cropped heads in photos, or slow pages caused by large images."
license: MIT
metadata:
  author: BrivaHamisi
---

# Image uploads: sized WebP, JPEG share images, sensible crops

One call turns an upload into every size the site needs:

```php
$path = app(ImageUploader::class)->store($request->file('image'), 'posts');
// posts/9b1e.../original.webp, card.webp, detail.webp, thumb.webp, share.jpg
$post->update(['image' => $path]);

$post->image_urls['card'];   // URL of the card size
$post->image_urls['share'];  // 1200 × 630 JPEG for og:image
```

## Install

1. `composer require intervention/image:^3.11`
2. Copy from `stubs/`, dropping `.stub`: `config/images.php`, `app/Support/ImageUploader.php`, `app/Models/Concerns/HasImageUrls.php`, `app/Console/Commands/RegenerateImages.php`, `tests/Feature/ImageUploaderTest.php`.
3. Run `php artisan storage:link` (for the default `public` disk).
4. Define sizes per type in `config/images.php`, named after where they're shown (`card`, `detail`, `thumb`, `share`). Match them to the actual CSS box at 1x–2x.
5. In each model with an image, add the `HasImageUrls` trait and an accessor:
   ```php
   protected function imageUrls(): Attribute
   {
       return Attribute::get(fn () => $this->resolveImageUrls($this->image, 'posts'));
   }
   ```
   Add `'image_urls'` to `$appends` if the model is sent to Inertia or an API.
6. **List every model in `config('images.models')`**, so `php artisan images:regenerate` rebuilds it after a size change. A model missing from that list silently keeps old sizes.
7. When replacing or deleting, call `$uploader->delete($oldPath)`. It removes the whole folder of sizes.

## What it gets right

- **WebP for the site, JPEG for previews.** WebP is 25–35% smaller, but WhatsApp, Facebook and LinkedIn often show no preview image for WebP. Sizes marked `'jpg'` (like `share`, 1200 × 630) are saved as JPEG.
- **Heads stay in the picture.** Cropping a portrait photo into a landscape box around the centre cuts off heads. A photo taller than the target keeps its **top**. The other photos are cropped around the centre. A test checks this with a two-colour image.
- **Logos are never cropped.** Sizes marked `'contain'` scale to fit inside the box.
- **No upscaling.** A 640-pixel upload stays 640 pixels as the original. Only the fixed-box sizes are filled.
- **The original is capped** at `max_width` (1920), so a 12 MB phone photo doesn't become a 12 MB page.
- **Regenerating doesn't degrade quality.** The original is never re-encoded. Older single-file uploads are converted into sized folders.
- **Bundled images** (paths under `assets/`) work with the same `image_urls` API: every size points at the one file.

## Rules

- Validate uploads: `['image', 'mimes:jpg,jpeg,png,webp,avif', 'max:10240']`. Resizing a huge image needs memory, so raise `memory_limit` for upload requests if needed.
- Output `<img src="{{ $post->image_urls['card'] }}" width="400" height="532" loading="lazy" decoding="async" alt="...">`. Explicit width and height stop layout shift. Don't lazy-load the image at the top of the page (the LCP image).
- Use `imagick` (`IMAGES_DRIVER=imagick`) when available: it's faster and keeps colour profiles. GD works everywhere.
- For S3, set `IMAGES_DISK=s3`. Everything goes through the `Storage` facade.
- Seeders: store each photo once. Check `ImageUploader::isUpload($model->image)` before uploading again, or every `db:seed` creates new folders.
