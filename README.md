# Timeline.md

A browser-only Vue app for turning simple Markdown into a visual timeline.

## Format

```md
# Product launch

## Act 1 — Discovery
- [A1] Research begins @ -2
- [A2] Direction approved @ 4
```

Positions may be negative or positive. Data is saved in `localStorage`; PNG export happens locally in the browser.

## Run

```bash
npm install
npm run dev
```
