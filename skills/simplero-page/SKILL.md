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
 Popups

 Every Simplero page has one built-in popup. Its content is an HTML section like any other,
 so you write it the same way — but closing it is NOT your job to reimplement.

 The popup is a real modal: it owns a full-screen overlay, Esc handling, click-outside-to-close,
 and video pausing. Hiding your own markup does not close it — it leaves the visitor staring at a
 dimmed, empty overlay that only Esc or a click outside can dismiss.

 NEVER write a close button like this:

   <!-- WRONG. Hides your content; the modal and its overlay stay open. -->
   <button onclick="this.closest('.my-card').style.display='none'">No thanks</button>

   <!-- Also wrong, for the same reason -->
   <button onclick="this.closest('.my-card').remove()">No thanks</button>

 Write it like this instead — a declarative Stimulus action, no inline JS:

   <button data-action="click->builder--popup#closePopup">No thanks</button>

 That runs exactly the same path as the popup's built-in X: closes the modal, hides the overlay,
 and stops any playing video. It works at any nesting depth inside the popup, and needs no script
 tag, no ids, and no selectors.

 To OPEN the popup from elsewhere on the page, put data-open-popup on any element:

   <button data-open-popup>Show me the offer</button>

 Both attributes only work in the right place: data-action="click->builder--popup#closePopup"
 does nothing unless the button is inside the popup, and data-open-popup belongs on the page
 outside it. If a close button seems to do nothing, check that it is actually inside the popup.

 ---

 You may use the playright tool to view how the page looks currently if given instructions to copy design from a specific other page.
 And then you can also use it to view the page and make any changes if you notice any thing funky.
 Note: the page must be published (publish: true) for it to be viewable in the browser.
