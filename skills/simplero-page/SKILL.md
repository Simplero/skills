---
name: simplero-page
version: 0.2.0
description: When user wants you to build a web page hosted on Simplero.
user-invocable: true
---


You're responsible for building pages in Simplero through the API.

When user asks you to build one, ask for their API key if you don't already have it, then use the following info:

Headers: `X-API-Key: <key>` and `User-Agent: AppName (email@example.com)`
Base URL: `https://simplero.com/api/v2`
API reference (source of truth): https://simplero.com/api/v2/docs — fetch this to discover available endpoints, request/response shapes, and node types.

---
## Step 1 — Ask which kind of page

Don't assume. Before doing anything, ask the user which kind of page this is:

- **Site page** — lives under one of their Sites. Has the site's header and footer wrapped around it. Use the `pages` API resource. Always create with `builder_doc: true` (themed pages are legacy).
- **Landing page** — standalone page, no site chrome. Use the `landing_pages` API resource. Always create with `page_type: 'builder_doc'` (themed pages are legacy).
- **Funnel page** — not supported via the API yet. Funnel pages would normally be created by adding a funnel step (which then creates the landing page for you), but that flow isn't exposed in the API. If the user wants one, tell them they need to add the step in the UI first; once the landing page exists you can edit it via the `landing_pages` resource.

If they pick a site page and you don't already know which site, ask which one. Use the API to list their sites if you need to give them options.

---
## Step 2 — Create the page

- Site page: `POST /pages` with `site_id`, `name`, `builder_doc: true`.
- Landing page: `POST /landing_pages` with `name`, `page_type: 'builder_doc'`.

Both return an `id` you'll use for all builder calls below.

**For site pages: do NOT include the site's header or footer in your HTML.** The site already wraps the page with them — adding your own will double them up. Build only the body content.

---
## Step 3 — Ask: HTML-only or structured builder tree?

Two ways to build content. Ask the user which they want and explain the tradeoff:

- **HTML-only (preferred for most pages, especially when building from scratch).** You write raw HTML for each section. Fast, flexible, fewer round-trips. Tradeoff: in the editor, the user can't drag/drop or tweak individual elements inside a section the way they can on a normal builder page — sections show as raw HTML blocks. Best for marketing pages, hero sections, copy-heavy pages, anything you're authoring end-to-end.
- **Structured node tree.** You build the page as a hierarchy: `body → section → row → column → element`, one node at a time via the API. The user gets full editor control over every element afterward. Tradeoff: many more API calls, slower, verbose. Best when the user wants editor parity, or when you're tweaking one setting on one element of an existing structured page.

You can also mix — start with HTML sections and embed structured nodes (order forms, opt-in inputs, buttons) inside them via the embed endpoint (covered below).

If the user later complains they "can't edit elements like on other pages," it's because the page was built with html_sections — explain it in friendly terms and offer to rebuild with the structured approach.

---
## Step 4a — Building with HTML sections

Create the page as **multiple html_section nodes, one per logical section**, each with a descriptive `name` (Hero, Features, Testimonials, Pricing, FAQ, CTA, etc.). One blob is much worse — separate named sections let the user re-arrange, edit, or replace individual sections in the editor without touching the rest.

For each section:

1. `POST /<resource>/:id/builder/content/html_section` with:
   - `node_id: "create_new"` (creates a new html_section under the page body)
   - `name: "Hero"` (descriptive)
   - `html: "<section>...</section>"`
2. Save the returned `node_id` if you'll need to update that section later. Always fetch with `GET .../builder/content/html_section?node_id=<id>` before updating, so you don't overwrite editor changes. **You must send full HTML on every update.**
3. If you don't know what html_section nodes already exist on a page, send an invalid/missing `node_id` — the error response includes `existing_html_section_node_ids`.

`<resource>` is `pages` for a site page, `landing_pages` for a landing/funnel page.

---
## Step 4b — Building with the structured node tree

Hierarchy: `body → section → row → column → element`. Build top-down.

1. Discover what nodes are addable here, optionally filtered by parent: `GET /<resource>/:id/builder/content/node_types?parent_node_id=<id>`
2. For each node type you'll use, inspect available settings: `GET .../node_types/detail?type=<type>`, then `GET .../node_types/settings_detail?type=<type>&setting_ids[]=<id>` to get arg shapes/defaults.
3. Create nodes one at a time: `POST /<resource>/:id/builder/content/nodes` with `type`, `parent_node_id`, optional `after_node_id` (for ordering), optional `name`, and `settings`.
4. Update settings: `PATCH /<resource>/:id/builder/content/nodes/:node_id` (settings are deep-merged — only pass what you want to change).
5. Delete: `DELETE /<resource>/:id/builder/content/nodes/:node_id`.
6. Always look up the actual settings schema before setting values — don't guess arg names.

---
## Embedding dynamic Simplero elements inside HTML

Inside an html_section, embed structured nodes (order forms, opt-in inputs, buttons, surveys, scheduling links, video, blog post lists, etc.):

1. Discover the type and its settings (same `node_types` endpoints as above).
2. `POST /<resource>/:id/builder/content/html_section/embed` with `type` and `settings`. Returns `embed_html` — a self-contained `<builder-node>...</builder-node>` snippet.
3. Paste `embed_html` verbatim into the section's HTML wherever you want the element to appear. At runtime it's replaced with the rendered node.

Avoid internal-only elements (e.g. Simplero trial signup) even if exposed.

---
## Forms and opt-ins

There is no "Form" element. The whole page acts as one form; the popup is a second form boundary. To collect submissions:

1. Add `Input` element nodes for each field you want to collect. Ask the user which fields if it's not obvious.
2. Each Input must be linked to a **FormField** via `form_field_id`. A FormField is an instance of an account-level `Field` attached to this builder doc.
   - First find the underlying `Field` (account-scoped, accounts usually have many already). Use the API to look it up; create one if it doesn't exist.
   - Then find an existing FormField for this builder doc with that field, or create one. The page's response includes a `content_builder_doc_id` you'll need as the FormField's `owner_id` (with `owner_type: "BuilderDoc"`).
   - Set the resulting form_field id on the Input's `form_field_id` setting.
3. Add a **Button** element with its action set to `opt_in`. This submits the form. Set additional opt_in args (e.g. `list_id` to subscribe to a list) as needed.

---
## Popups

Every page has a popup node — opens content in a modal. It uses the same hierarchy as the body (section > row > column > element, OR html_section). A button's action can be set to open the popup.

- **Never create or delete the popup node itself.** It's already there.
- You can modify its settings (auto show/hide, position, etc.) and add/update/delete content inside it as children.
- To put HTML in the popup: create an html_section with `POST .../builder/content/nodes` (type=`html_section`, `parent_node_id`=popup's node id, no settings on create), then call the html_section update endpoint with the new node's id to set its HTML.

---
## Step 5 — Publish

Changes land in a working version. To go live:

- Pass `publish_changes: true` on any mutating builder call to publish inline, **or**
- Call `POST /<resource>/:id/builder/content/publish` once at the end.

---
## Notes

- Always consult https://simplero.com/api/v2/docs for the full, current endpoint list and schemas — that's the source of truth.
- Record IDs (product_id, survey_id, list_id, field_id, etc.) needed for node settings aren't always listable via the API yet. If you can't find one, ask the user or have them look it up.
- You may use the playwright tool to view how the page looks (or copy a design from a reference page) and to verify the result after changes.
