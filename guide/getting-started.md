---
title: Getting started
---

# Getting started

This page is a markdown file pulled from the content repo. Everything here is standard GFM.

## Code highlighting

```ts
export function greet(name: string): string {
	return `hello, ${name}`;
}
```

## Tables

| File          | Becomes          |
| ------------- | ---------------- |
| `foo.md`      | `/foo`           |
| `docs/bar.md` | `/docs/bar`      |
| `a/index.md`  | `/a`             |

## Lists

1. Edit markdown, push to the repo.
2. The app pulls every 5 minutes.
3. The tree and pages update on the next request.

> Files and folders starting with `.` or `_` are hidden from the tree, and frontmatter `draft: true` pages are only visible to admins.
