---
name: inertia-admin-tables
description: "Add pagination to Laravel + Inertia React admin tables with numbered pages, a row count, and a rows-per-page selector (shadcn/ui) that keeps search and filter query strings, backed by a server-side allow-list. Use when building admin or dashboard tables, paginating Inertia pages, adding per-page selectors, or when pagination is missing, loses filters, or a shadcn Select opens with nothing showing."
license: MIT
metadata:
  author: BrivaHamisi
---

# Admin tables with real pagination (Laravel + Inertia React + shadcn/ui)

Server: `->paginate($this->perPage())->withQueryString()`. Client: `<DataPagination paginator={posts} label="posts" />`. The footer shows:

- "Showing 1–25 of 117 posts". It's always visible, so admins know the table is complete.
- Numbered pages with "…" gaps, from Laravel's own `links`. Phones show "Page 2 of 5" instead.
- A **Rows per page** selector (10, 25, 50, 100), shown once there are more than 10 rows.

## Install

1. Copy from `stubs/`, dropping `.stub`:
   - `app/Http/Controllers/Concerns/PaginatesTables.php`
   - `resources/js/components/data-pagination.tsx`
   - `resources/js/types/paginated.ts`
   - `tests/Feature/PaginatesTablesTest.php`
2. The component needs the shadcn/ui `button`, `pagination` and `select` components: `npx shadcn@latest add button pagination select`.
3. In each admin controller:
   ```php
   use App\Http\Controllers\Concerns\PaginatesTables;

   class PostController extends Controller
   {
       use PaginatesTables;

       public function index(Request $request): Response
       {
           return Inertia::render('admin/posts/index', [
               'posts' => Post::query()->latest()->paginate($this->perPage())->withQueryString(),
           ]);
       }
   }
   ```
4. Under each table: `<DataPagination paginator={posts} label="posts" />`, typed as `posts: Paginated<Post>`.

## Rules and gotchas

- **`withQueryString()` on every paginator.** Without it, page links drop `?q=` and `?status=`, and page 2 of a search shows unfiltered results.
- **Allow-list page sizes on the server.** `per_page=100000` must not dump a whole table. `perPage()` accepts only `PER_PAGE_OPTIONS` and falls back to 25. Keep the PHP and TSX lists identical.
- **Changing the page size goes back to page 1.** Otherwise page 5 of 10-row pages becomes an empty page of 100-row pages. The component removes `page` from the URL.
- **Selects at the bottom of the page must open upwards.** Many shadcn `SelectContent` setups use `position="popper"`, `side="bottom"` and `avoidCollisions={false}`, and cap their height to the space below the trigger. In a table footer that space is close to zero, so **the list opens with nothing showing**. The component passes `side="top" avoidCollisions`.
- **Don't hide the footer when everything fits.** "Showing 1–12 of 12" confirms nothing is missing. Only the page links hide on a single page.
- **Stale tabs after deploys:** Inertia reloads the page when the asset version changes, but only if `HandleInertiaRequests::version()` returns the Vite manifest hash (the default). If clicks do nothing after a deploy, check the browser console for "Failed to fetch dynamically imported module".
- **Big tables:** `paginate()` runs a `count(*)` on every page. For millions of rows, use `simplePaginate()` or `cursorPaginate()` and hide the total.
