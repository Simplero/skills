---
name: simplero-page
version: 0.1.0
description: When user wants you to build a web page hosted on Simplero.
user-invocable: true
---


You're responsible for building landing pages in simplero through the api. 

When user asks you to build one, ask for their API key if you don't already have it, then use the following info:

Headers: X-API-Key: <key> and User-Agent: AppName (email@example.com)
Base URL: https://simplero.com/api/v1

 ---
 Typical Workflow

 1. Get embed codes for any elements you need — POST /builder_doc_nodes/embed with type and settings. Types include elements/order_form, elements/survey, elements/scheduling_link, elements/video, etc. Returns a <builder-node> HTML tag you drop into your page.
 2. Build your HTML — write your page HTML and place the <builder-node> embed tags wherever you want the elements to appear.
 3. Create the page — POST /landing_pages with name and html. Returns the page ID and URL.
 4. View/update later — GET /landing_pages/:id to fetch current HTML, PATCH /landing_pages/:id with html to update. Always fetch first so you don't overwrite editor changes.
 5. To refresh embeds — fetch the page, strip old embed tags, get fresh embed codes from /builder_doc_nodes/embed, re-insert them, and PATCH.

 ---
 Discovering Available Elements

 - GET /builder_doc_nodes/ — lists all embeddable element types grouped by category
 - GET /builder_doc_nodes/elements/order_form (or any type) — shows available settings and their defaults

 ---
 Landing Pages API

 POST /landing_pages — Create a landing page
   params: name (string, optional, defaults to "Untitled"), html (string, required), publish (boolean, optional, defaults to false)
   Returns: { id, name, url, html, active }
   The page is not published unless you pass publish: true. Without publishing, the page exists as a draft.

 GET /landing_pages/:id — Get a landing page
   Returns: { id, name, url, html, active }

 PATCH /landing_pages/:id — Update a landing page
   params: name (string, optional), html (string, optional), publish (boolean, optional, defaults to false)
   Pass publish: true to publish it immediately.
   Returns: { id, name, url, html, active }

 ---

 You may use the playright tool to view how the page looks currently if given instructions to copy design from a specific other page.
 And then you can also use it to view the page and make any changes if you notice any thing funky.
 Note: the page must be published (publish: true) for it to be viewable in the browser.
