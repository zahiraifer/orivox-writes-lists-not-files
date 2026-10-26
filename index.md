# Why Our AI Website Editor Writes Lists, Not Files

*By Zahi Raifer*

Most people picture an AI website builder as a model that rewrites your page every time you ask for something. You type "make the button green," and the model hands back a fresh copy of the whole page with, you hope, only the button changed. That is the design we deliberately avoided at [Orivox](https://orivox.org). This piece explains the one decision that shapes everything else in our editor: the model never touches a file. It only writes a short list of instructions, and ordinary Python code carries them out.

## The problem with "just regenerate it"

Asking a language model to return a whole page after every request has a quiet failure mode. The model does not just make your change. It retypes everything else, and every retyped line is a chance for drift: a sentence shortened, a phone number dropped, a logo swapped for a placeholder, a legal notice next to a contact form gone. Small business owners rarely read their HTML. They notice weeks later, when a customer can't find the WhatsApp link.

We wanted the opposite guarantee. When an owner asks for one change, they get exactly one change and nothing else on the page moves.

## Giving every element an address

The first step happens when a site is built. After the model drafts the initial page in HTML and Tailwind, Python walks the whole document and gives each element a short, unique `data-ai-id`. From then on, every edit points at an element by that id, for example `[data-ai-id='a0f0']`.

That sounds like a small detail, but it changes the nature of the conversation. The model no longer has to describe *where* something is in prose, or reproduce the surrounding markup to locate it. It can just name the element, the way you would point at it on a screen.

Shared parts like the header, footer and head scripts are stored once as partials, so a footer fix reaches every page in one move. The original site is committed to Git as a baseline before the first edit, so the owner can always go back to what they started with.

## A vocabulary of fourteen verbs

Instead of writing pages, the planner writes operations from a fixed grammar of 14 verbs: `set_text`, `replace_html`, `insert_before`, `insert_after`, `append_html`, `prepend_html`, `remove`, `set_attr`, `remove_attr`, `add_class`, `remove_class`, `move_before`, `move_after` and `move_to_page`.

A request like "make the FAQ cards dark" becomes a handful of class changes on the card elements. A request like "move the testimonials above pricing" becomes a single `move_before`. The answers are tiny: across 24 benchmark runs, the median planner output was 141 tokens. That's a few operations, not a rewritten page.

A list like this has properties a blob of HTML does not. It can be validated before anything is saved. It can be logged, so we can see exactly what the model proposed. And it can be replayed, which turns out to be the foundation of how we test.

## What one turn looks like

When an owner types a request in Hebrew or English, the web app checks the account, queues the turn and takes a lock on the project. The editing service assembles the page in memory, partials inlined, and asks the planner what to do.

The planner can answer in words or call one of 11 tools. Besides the main edit tool, it can read another page, look at a screenshot of the rendered result, read a page the owner links to (with their consent), pull a palette from an image, generate an image, create a page, rewrite a page, or restore an earlier version. Tools that gather information feed their result back so the planner can keep going, within a fixed budget of six model calls. An edit call ends the turn.

Then Python applies the operations with BeautifulSoup, runs its checks, saves, commits to the project's Git repository and records the turn. The browser reloads the preview with the change in place. Every change is a commit, so every change is reversible.

## Raw HTML beat the clever summary

We expected a compact JSON summary of the page to be a cheaper and cleaner input for the planner than raw HTML. We benchmarked both on real edit requests, and raw HTML won: the same understanding at about half the tokens, with the planner seeing every element. We now send raw HTML by default and fall back to the summary only for very large pages.

A typical request is about 42,000 input tokens: roughly 23,000 for the editing rules, 3,500 for the tool definitions, a median of about 16,000 for the page itself, and a little over a thousand for recent conversation and the request. The rules and tool definitions don't change between turns, so they're cached, which keeps the fixed part of every request fast and inexpensive.

## Guardrails that don't trust the model

The operation grammar narrows what the model *can* do. Deterministic checks in Python decide what actually gets saved, and they hold no matter what the model proposes:

- A save may never empty a live page.
- Moving or splitting a paragraph must keep every sentence.
- A whole-page rewrite must keep every section the owner didn't mention.
- Image addresses are re-checked on every save so images keep loading at the right size.
- The business's real logo can't be swapped for a placeholder.
- The contact form's privacy notice stays next to the form.
- The assistant can't claim success unless something actually changed.
- A page must be read before another page's edit is based on it.

Each check is pinned by a replay test. Because turns are recorded as operation lists, we can replay real past turns against new code and confirm a guardrail still holds. Under all of this are two rules: only the server decides whether a turn is saved, and the model's word is never taken as proof that something happened.

## The trade-off we accepted

A narrow grammar is less flashy than "the AI rebuilds your site." Some requests take the planner a couple of tool calls instead of one big rewrite, and we had to design the vocabulary carefully. In return, owners get edits they can predict, history they can undo, and a site that doesn't slowly erode with each conversation. For a small business whose website is its storefront, we think that's the right trade.

The broader lesson is one we'd offer to anyone building with language models: let the model decide *what* to do, and let plain, testable code decide *how* it's done.

---

**About the author**

Zahi Raifer is the founder of Orivox, an AI website builder for small businesses that lets owners create and edit their sites by chatting in Hebrew or English.

*AI disclosure: This article was drafted with the help of an AI writing assistant, based on Orivox's internal technical documentation, and reviewed and edited by the author.*
