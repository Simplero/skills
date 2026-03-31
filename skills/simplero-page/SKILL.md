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
 5. To refresh embeds — fetch the page, strip old <builder-node> tags, get fresh embed codes from /builder_doc_nodes/embed, re-insert them, and PATCH.

 Avoid using internal elements like Simplero trial signup, even if it's available through the API.

 ---
 Discovering Available Elements

 - GET /builder_doc_nodes/ — lists all embeddable element types grouped by category
 - GET /builder_doc_nodes/elements/order_form (or any type) — shows available settings and their defaults
 - Record IDs (product_id, survey_id, etc.) aren't yet available via the API — use rails runner to look them up, then pass them to the embed endpoint

 You may use the playright tool to view how the page looks currently if given instructions to copy design from a specific other page.
 And then you can also use it to view the page and make any changes if you notice any thing funky
