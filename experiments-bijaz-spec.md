# Build Spec: "experiments with bijaz" — Static Blog Site

You are building a minimal static blog site called "experiments with bijaz." It is a lab notebook where a developer documents small experiments with an AI coding agent called bijaz.

## Architecture

- Single `index.html` file with inline CSS and JS
- Posts stored as separate `.md` files in a `/posts` folder
- A `posts.json` config file that lists all posts (title, date, filename, number)
- Uses `marked.js` (CDN) to render markdown
- Client-side routing (hash-based, e.g. `#post/1`) — no page reloads, no build step, no framework

## File Structure

```
/
├── index.html
├── posts.json
├── posts/
│   └── 01-building-my-own-agent.md
└── images/
│   └── header-title.png (the "experiments with bijaz" heading as an image)
└── (any post images go here too)
```

## posts.json Format

```json
[
  {
    "number": 1,
    "title": "building my own agent",
    "date": "25.02.2026",
    "file": "01-building-my-own-agent.md"
  },
  {
    "number": 2,
    "title": "what happens if i remove the system prompt",
    "date": "26.02.2026",
    "file": "02-removing-the-system-prompt.md"
  },
  {
    "number": 3,
    "title": "why is token usage so high",
    "date": "27.02.2026",
    "file": "03-token-usage.md"
  }
]
```

Posts are displayed in reverse order (newest first) on the homepage. The `number` field is the experiment number, NOT the display order.

## Design Spec

### Global

- Background color: `#FFFFFF` for the content area
- Body/outer background: `#1a1a1a` (dark frame visible on desktop around the white content card)
- The white content area should be a centered card/container with max-width ~700px, min-height 100vh, with some horizontal padding
- All text is lowercase in spirit (the content is written lowercase, but don't force CSS lowercase — just keep the UI text lowercase)
- No emojis, no icons (except a back arrow on post pages)
- No hamburger menus, no footer, no sidebar, no social links. Absolutely nothing extra.

### Typography

- **Heading ("experiments with bijaz"):** Render as an image (`header-title.png`). This avoids font availability issues across platforms. The image should be sized appropriately for both desktop and mobile. On mobile it should be smaller. Use an `<img>` tag with appropriate alt text. For now, as a placeholder until the real image is provided, render the heading using Google Font "Nanum Pen Script" (import from Google Fonts) as a fallback — "experiments" in large size, "with bijaz" in smaller size, offset/indented to the right on the line below. Style it to look handwritten. When the real image is ready, it replaces this.
- **Quote:** Roboto Mono, italic, grey (`#666`), centered below the heading. The quote is: `"i don't speak", bijaz said. "i operate a machine called language. it creaks and groans, but is mine own."`
- **Body / post list / post content:** Roboto Mono, 400 weight, `#000` for primary text
- **Dates:** Roboto Mono, 400 weight, `#999`
- Import Roboto Mono and Nanum Pen Script from Google Fonts

### Homepage Layout

1. **Header section** — takes up roughly the top 50-60% of the viewport on first load (generous whitespace, content pushed low). Contains:
   - The heading image/text centered
   - The bijaz quote below it, centered
   
2. **Post list** — sits in the lower portion of the page. Each row:
   - Left side: `{number}. {title}` in Roboto Mono, black
   - Right side: `{date}` in Roboto Mono, grey
   - Use flexbox with space-between for alignment
   - Thin divider line (`1px solid #ddd`) between each row AND above the first row
   - Each row is a clickable link (cursor pointer, no underline). No hover color change needed — keep it minimal, but a subtle opacity change on hover is fine.
   - Rows are listed newest-first (highest number on top)
   - The list is NOT in a scroll container — the page itself scrolls naturally when the list gets long

3. **Responsive behavior:**
   - On mobile (< 600px): tighter padding, smaller heading image/font, the layout should match the iPhone mockup — same structure, just tighter
   - On desktop: the white card is centered with dark background visible on sides, matching the MacBook mockup

### Post Page Layout

1. **Header (persistent across pages):**
   - The same "experiments with bijaz" heading image/text at the top, same position as homepage
   - Clicking it navigates back to the homepage
   - No quote on post pages — just the heading

2. **Post header:**
   - A back arrow `←` or `← back` link above or to the left of the post title, in grey, clicking returns to homepage
   - Post title displayed as: `{number}. {title}` in Roboto Mono, black, slightly larger than body text
   - Date below the title in Roboto Mono, grey
   - Thin divider line below the date before content begins

3. **Post content (rendered markdown):**
   - Paragraphs: Roboto Mono, regular weight, `#000`, comfortable line-height (~1.7)
   - Images: full-width of the content column, with some vertical margin. No borders, no shadows. Just the image.
   - Code blocks: if any appear in markdown, style them with a light grey background (`#f5f5f5`), some padding, Roboto Mono (already the body font so it'll match)
   - Links: black with underline
   - Headings (h2, h3 if used): Roboto Mono, semi-bold, slightly larger than body
   - Keep generous spacing between elements — it should breathe

4. **Responsive:** Same behavior as homepage — content card narrows on mobile, images scale down

## Routing

- `#` or empty hash → Homepage
- `#post/{number}` → Post page for that experiment number
- Use `hashchange` event listener
- When navigating to a post, fetch the corresponding `.md` file from `/posts/`, render it with `marked.js`, and display it in the post page layout
- When navigating back to home, show the homepage layout
- Scroll to top on navigation

## Sample Post (create this as a placeholder)

Create `posts/01-building-my-own-agent.md` with this content:

```markdown
i built a coding agent in a day. ~250 lines of python. it calls the claude api with 4 tools — bash, read_file, write_file, edit_file — and runs an autonomous loop until the task is done.

the build process was modular. first a basic api call. then a conversation loop. then tool definitions as json schemas. then executors for each tool. then the agent loop that ties it all together. finally, polish — error handling, cost tracking, colored output, ctrl+c handling.

the core loop is ~80 lines of real logic. that surprised me. the agent receives a prompt, calls claude, parses tool calls from the response, executes them, feeds results back, and repeats. that's it.

what actually matters is everything around that loop. token usage is the real challenge — my agent used ~86k input tokens for a simple api-fetching task because it resends the full conversation history every call. that's quadratic growth. system prompt wording directly affects this — adding "don't explore unless needed" saved significant tokens.

the things my agent doesn't have: streaming, context compaction, tool output truncation, session persistence, a tui. these are what separate a prototype from a product like claude code.

bijaz is ~5% of claude code's codebase but covers the core agent loop. the other 95% is efficiency, ux, and tool sophistication.

this is experiment 1. the prototype exists. now i want to poke at it and see what breaks.
```

Also create placeholder entries in `posts.json` for all 3 posts listed earlier (posts 2 and 3 don't need actual `.md` files yet — the site should handle missing files gracefully with a "post not found" message).

## Important Implementation Notes

- Keep the code clean and readable. This is a small site — no over-engineering.
- No build tools, no npm, no bundler. Just HTML, CSS, JS, and markdown files.
- Test that the routing works: homepage shows post list, clicking a post shows the post, clicking heading or back arrow returns to homepage.
- The site should look good at both viewport sizes shown in the mockups: ~375px (iPhone) and ~1440px (MacBook Air).
- If `marked.js` CDN fails to load, the site should still be functional (show raw markdown as fallback).
- All text content in the UI should be lowercase (titles, navigation, etc.)
