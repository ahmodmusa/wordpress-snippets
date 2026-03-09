# 🪝 Hooks — Actions & Filters

> Everything you need to know about WordPress hooks.

---

## Action vs Filter — Quick Difference

| | Action | Filter |
|---|---|---|
| Purpose | Do something | Modify something |
| Return value | Not required | **Required** |
| Function | `add_action()` | `add_filter()` |
| Trigger | `do_action()` | `apply_filters()` |

```php
// Action — run code at a point
add_action( 'wp_footer', function() {
    echo '<p>Added to footer!</p>';
});

// Filter — modify a value and return it
add_filter( 'the_title', function( $title ) {
    return strtoupper( $title ); // Must return!
});
```

---

## Remove Default Hooks

```php
// Remove a hook added with a named function
remove_action( 'wp_head', 'wp_generator' );       // Remove WP version
remove_action( 'wp_head', 'wlwmanifest_link' );   // Remove Windows Live Writer
remove_action( 'wp_head', 'rsd_link' );           // Remove RSD link

// Remove a hook added with a class method
// Must use same priority as when it was added!
remove_action( 'init', [ $object, 'method_name' ], 10 );
```

---

## Common Hook Reference

### Page Load Order
```
muplugins_loaded → plugins_loaded → setup_theme → after_setup_theme
→ init → wp_loaded → template_redirect → wp_head → [content] → wp_footer
```

### Admin Hook Order
```
plugins_loaded → init → admin_init → admin_menu → admin_enqueue_scripts → admin_head
```

---

## Create Your Own Hook

```php
// In your plugin/theme — define the hook
function my_plugin_process_data( array $data ): array {
    // Allow others to modify $data before processing
    $data = apply_filters( 'my_plugin_before_process', $data );

    // Do processing...

    // Allow others to run code after
    do_action( 'my_plugin_after_process', $data );

    return $data;
}

// Other developers can now extend your plugin:
add_filter( 'my_plugin_before_process', function( $data ) {
    $data['extra'] = 'value';
    return $data;
});
```

---

## Priority Guide

```php
// Default priority is 10
add_action( 'init', 'my_function' );        // Priority 10

// Run earlier (lower number)
add_action( 'init', 'my_function', 5 );

// Run later (higher number)
add_action( 'init', 'my_function', 20 );

// Run very late (useful for overriding plugins)
add_action( 'init', 'my_function', 9999 );
```
