# 🤝 Contributing to WordPress Developer Snippets

First off — thank you! Every snippet you add helps thousands of developers save time. Contributions of all sizes are welcome. 🎉

---

## 📋 Table of Contents

- [How to Contribute](#how-to-contribute)
- [Snippet Categories](#snippet-categories)
- [Snippet Format](#snippet-format)
- [Quality Rules](#quality-rules)
- [Commit Message Format](#commit-message-format)
- [Code Style](#code-style)
- [What Makes a Great Snippet?](#what-makes-a-great-snippet)

---

## How to Contribute

### Step 1 — Fork & Clone
```bash
git clone https://github.com/YOUR_USERNAME/wordpress-snippets.git
cd wordpress-snippets
```

### Step 2 — Create a Branch
Use a descriptive branch name:
```bash
git checkout -b snippet/ajax-file-upload
git checkout -b fix/security-nonce-example
git checkout -b update/hooks-readme
```

### Step 3 — Add Your Snippet
Find the right category folder under `snippets/` and open its `README.md`.

Add your snippet at the bottom following the [format below](#snippet-format).

### Step 4 — Commit & Push
```bash
git add snippets/ajax/README.md
git commit -m "Add: AJAX file upload with validation"
git push origin snippet/ajax-file-upload
```

### Step 5 — Open a Pull Request
Go to [github.com/ahmodmusa/wordpress-snippets](https://github.com/ahmodmusa/wordpress-snippets) and open a Pull Request against the `main` branch. Fill in the PR template.

---

## Snippet Categories

| Folder | What belongs here |
|---|---|
| `snippets/hooks/` | Actions, filters, hook utilities |
| `snippets/queries/` | WP_Query, get_posts, $wpdb, terms, users |
| `snippets/security/` | Sanitization, escaping, nonces, capabilities |
| `snippets/performance/` | Caching, asset loading, query optimisation |
| `snippets/gutenberg/` | Blocks, patterns, FSE, block.json, theme.json |
| `snippets/ajax/` | AJAX handlers (PHP + JS) |
| `snippets/rest-api/` | Custom endpoints, authentication, filters |
| `snippets/theme/` | Theme setup, customizer, menus, template parts |
| `snippets/plugin/` | Plugin boilerplate, settings, admin, shortcodes |

Not sure where your snippet belongs? Open an issue and ask — we'll help.

---

## Snippet Format

Every snippet should follow this structure inside the category `README.md`:

~~~markdown
## Snippet N — Short Descriptive Title

Brief one or two sentence description of what this does and when to use it.

```php
// Well-commented, working code here
add_action( 'init', 'my_example_function' );
function my_example_function(): void {
    // Your snippet
}
```

**When to use:** One sentence on the ideal use case.
~~~

### Rules for the format

- Title should be clear and searchable (e.g. "Limit Login Attempts", not "Login Stuff")
- Description must explain the **why**, not just the **what**
- Code must be complete and copy-paste ready
- Add `// comments` where the logic is not immediately obvious
- "When to use" should be genuinely useful — not just repeat the title

---

## Quality Rules

Before submitting, verify your snippet:

- ✅ **Tested** — code works on WordPress 6.0+ and PHP 8.0+
- ✅ **Sanitized** — all user input uses `sanitize_*()` or `absint()` etc.
- ✅ **Escaped** — all output uses `esc_html()`, `esc_attr()`, `esc_url()` etc.
- ✅ **Unique** — snippet is not already in the repo (search before adding)
- ✅ **No external dependencies** — unless clearly noted
- ✅ **No modification of WordPress core files**
- ✅ **Follows WordPress coding standards** (tabs, not spaces; Yoda conditions)
- ✅ **No hardcoded URLs** — use `home_url()`, `plugins_url()`, etc.

---

## Commit Message Format

```
Add: short description of what was added
Fix: what was wrong and what was corrected
Update: what changed and why
Remove: what was removed and why
```

Examples:
```
Add: AJAX rate limiting with transients
Fix: nonce verification in save_post example
Update: WP_Query pagination snippet to use paginate_links
```

---

## Code Style

Follow the [WordPress PHP Coding Standards](https://developer.wordpress.org/coding-standards/wordpress-coding-standards/php/).

Key rules:

```php
// ✅ Tabs for indentation (not spaces)
function my_function(): void {
    $value = 'hello';
}

// ✅ Single quotes for strings (unless interpolation needed)
$text = 'Hello World';

// ✅ Space before opening parenthesis on control structures
if ( $condition ) {
    // ...
}

// ✅ Yoda conditions (value on left)
if ( 'publish' === $post->post_status ) {
    // ...
}

// ✅ Type declarations on functions (PHP 8.0+)
function my_function( int $id, string $title ): array {
    // ...
}

// ✅ Return type void when nothing is returned
function my_action_callback(): void {
    // ...
}
```

---

## What Makes a Great Snippet?

The best snippets are ones that:

1. **Solve a real pain point** you've Googled more than once
2. **Demonstrate a best practice** not everyone knows
3. **Are hard to find** in the official docs
4. **Save meaningful time** — not just a one-liner anyone knows
5. **Are safe by default** — sanitized, escaped, nonce-verified

Snippets that get merged fastest:
- Real-world patterns (not just "hello world" examples)
- Security-focused patterns
- Performance tricks
- Things that work differently than you'd expect

---

## Questions?

Open an [issue](https://github.com/ahmodmusa/wordpress-snippets/issues) and ask — no question is too small!

You can also reach the maintainer directly:
- 🌐 [ahmodmusa.com](https://ahmodmusa.com)
- 💼 [Fiverr](https://www.fiverr.com/s/bd665oX)

---

Thank you for making this repo better for everyone! 🙌
