# Contributing to WordPress Snippets

Thank you for wanting to contribute! 🎉 Every snippet you add helps thousands of developers.

## How to Contribute

### Adding a New Snippet

1. Find the right category folder under `snippets/`
2. Open the `README.md` in that folder
3. Add your snippet following this format:

```markdown
### Short descriptive title

Brief description of what this does and when to use it.

```php
// Your well-commented code here
```

**When to use:** One line on the best use case.
```

### Snippet Quality Rules

- ✅ Code must be tested and working
- ✅ Must be compatible with WordPress 6.0+
- ✅ Follow WordPress coding standards
- ✅ Include a comment if logic isn't immediately obvious
- ✅ Use `sanitize_*`, `esc_*`, and nonces where relevant
- ❌ No external dependencies unless clearly noted
- ❌ No code that modifies the WordPress core files

### Commit Message Format

```
Add: short description of snippet
Fix: what was broken
Update: what changed and why
```

## Code Style

Follow the [WordPress PHP Coding Standards](https://developer.wordpress.org/coding-standards/wordpress-coding-standards/php/).

Key rules:
- Tabs for indentation (not spaces)
- Single quotes for strings unless interpolation needed
- Space before opening parenthesis on control structures: `if ( $x )`
- Yoda conditions: `if ( 'value' === $var )`

## What Makes a Great Snippet?

The best snippets are ones that:
1. Solve a problem you've Googled more than once
2. Are harder to find in the official docs
3. Demonstrate a best practice not everyone knows
4. Save meaningful time

## Questions?

Open an issue and ask — no question is too simple!
