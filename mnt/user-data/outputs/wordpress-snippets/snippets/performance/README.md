# ⚡ Performance Snippets

> Make your WordPress site fast. These snippets tackle the most common performance bottlenecks.

---

## Transients — Cache Expensive Operations

```php
/**
 * Generic cache-aside helper using transients.
 *
 * @param string   $key      Unique transient key.
 * @param callable $callback Function that returns the data to cache.
 * @param int      $expiry   Seconds until expiration. Default: 1 hour.
 */
function get_or_set_transient( string $key, callable $callback, int $expiry = HOUR_IN_SECONDS ): mixed {
    $cached = get_transient( $key );

    if ( false !== $cached ) {
        return $cached;
    }

    $data = $callback();
    set_transient( $key, $data, $expiry );

    return $data;
}

// Usage
$posts = get_or_set_transient( 'featured_posts_v1', function() {
    return get_posts([
        'meta_key'   => '_is_featured',
        'meta_value' => '1',
        'numberposts' => 5,
    ]);
}, DAY_IN_SECONDS );
```

---

## Invalidate Transient When Post Saves

```php
// Always clear related transients when content changes
add_action( 'save_post', 'clear_post_related_transients' );
add_action( 'deleted_post', 'clear_post_related_transients' );
function clear_post_related_transients( int $post_id ): void {
    delete_transient( 'featured_posts_v1' );
    delete_transient( 'homepage_hero_data' );
}
```

---

## Defer Non-Critical Scripts

```php
add_filter( 'script_loader_tag', 'defer_non_critical_scripts', 10, 3 );
function defer_non_critical_scripts( string $tag, string $handle, string $src ): string {
    $defer_scripts = [ 'my-analytics', 'my-chat-widget', 'my-social-share' ];

    if ( in_array( $handle, $defer_scripts, true ) ) {
        return str_replace( ' src=', ' defer src=', $tag );
    }

    return $tag;
}
```

---

## Lazy Load Images (Native HTML + WP Filter)

```php
// WordPress 5.5+ adds loading="lazy" by default.
// For older setups or to force it everywhere:
add_filter( 'wp_get_attachment_image_attributes', function( $attr ) {
    $attr['loading'] = 'lazy';
    return $attr;
});

// Set fetchpriority="high" for the LCP (hero) image
add_filter( 'wp_get_attachment_image_attributes', function( $attr, $attachment, $size ) {
    if ( is_singular() && get_post_thumbnail_id() === $attachment->ID ) {
        $attr['fetchpriority'] = 'high';
        $attr['loading']       = 'eager'; // Don't lazy-load LCP image!
    }
    return $attr;
}, 10, 3 );
```

---

## Limit Post Revisions

```php
// Add to wp-config.php
define( 'WP_POST_REVISIONS', 5 ); // Keep only last 5 revisions

// Or disable completely (not recommended for production)
define( 'WP_POST_REVISIONS', false );
```

---

## Remove Query Strings from Static Assets

```php
add_filter( 'script_loader_src', 'remove_query_strings_from_assets', 15 );
add_filter( 'style_loader_src', 'remove_query_strings_from_assets', 15 );
function remove_query_strings_from_assets( string $src ): string {
    $parts = explode( '?ver', $src );
    return $parts[0];
}
```

---

## Optimize WP_Query — Always Specify What You Need

```php
// ❌ Slow — fetches everything
$query = new WP_Query([ 'post_type' => 'post', 'posts_per_page' => 10 ]);

// ✅ Fast — only what you need
$query = new WP_Query([
    'post_type'              => 'post',
    'posts_per_page'         => 10,
    'no_found_rows'          => true,  // Skip COUNT query (only if you don't need pagination)
    'update_post_meta_cache' => false, // Skip if you don't need post meta
    'update_post_term_cache' => false, // Skip if you don't need terms
    'fields'                 => 'ids', // Only get IDs if you just need IDs
]);
```

---

## Heartbeat API — Reduce Server Load

```php
// Slow down or disable the WP Heartbeat API
add_filter( 'heartbeat_settings', function( $settings ) {
    $settings['interval'] = 60; // Check every 60 seconds (default: 15)
    return $settings;
});

// Disable heartbeat on frontend only
add_action( 'init', function() {
    if ( ! is_admin() ) {
        wp_deregister_script( 'heartbeat' );
    }
});
```

---

## Preconnect to External Origins

```php
add_action( 'wp_head', 'preconnect_external_origins', 1 );
function preconnect_external_origins(): void {
    $origins = [
        'https://fonts.googleapis.com',
        'https://fonts.gstatic.com',
    ];

    foreach ( $origins as $origin ) {
        printf( '<link rel="preconnect" href="%s" crossorigin>' . "\n", esc_url( $origin ) );
    }
}
```
