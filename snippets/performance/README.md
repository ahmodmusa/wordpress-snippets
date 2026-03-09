# ⚡ Performance

> 25 snippets to make your WordPress site faster — caching, query optimisation, asset loading, and more.

---

## Snippet 1 — Transient Cache: The Right Pattern

```php
/**
 * Cache-aside pattern using transients.
 * Always use this structure for expensive operations.
 */
function get_latest_news(): array {
    $cache_key = 'latest_news_v1';
    $cached    = get_transient( $cache_key );

    if ( false !== $cached ) {
        return $cached; // Return early — no DB hit
    }

    // Only runs on cache miss
    $posts = get_posts([
        'post_type'      => 'post',
        'posts_per_page' => 10,
        'post_status'    => 'publish',
        'no_found_rows'  => true,
    ]);

    set_transient( $cache_key, $posts, HOUR_IN_SECONDS );

    return $posts;
}

// Invalidate when content changes
add_action( 'save_post_post', fn() => delete_transient( 'latest_news_v1' ) );
add_action( 'deleted_post',   fn() => delete_transient( 'latest_news_v1' ) );
```

---

## Snippet 2 — Object Cache Helper

```php
/**
 * Get from object cache or run the callback and cache the result.
 * Object cache is faster than transients — stored in memory (Redis/Memcached).
 *
 * @param string   $key      Cache key.
 * @param string   $group    Cache group.
 * @param callable $callback Function that returns the data.
 * @param int      $expiry   Seconds until expiration.
 */
function get_or_cache( string $key, string $group, callable $callback, int $expiry = 3600 ): mixed {
    $cached = wp_cache_get( $key, $group );

    if ( false !== $cached ) {
        return $cached;
    }

    $data = $callback();
    wp_cache_set( $key, $data, $group, $expiry );

    return $data;
}

// Usage
$menu_items = get_or_cache( 'primary_menu', 'my_theme', function() {
    return wp_get_nav_menu_items( 'primary' );
}, DAY_IN_SECONDS );
```

---

## Snippet 3 — Optimised WP_Query (Skip What You Don't Need)

```php
// Every flag you add removes an unnecessary SQL query or JOIN
$args = [
    'post_type'              => 'post',
    'posts_per_page'         => 10,
    'post_status'            => 'publish',

    // Performance flags
    'no_found_rows'          => true,  // Skip COUNT(*) — only use when pagination not needed
    'update_post_meta_cache' => false, // Skip meta cache — only use when you don't need post meta
    'update_post_term_cache' => false, // Skip term cache — only use when you don't need terms
    'fields'                 => 'ids', // Return IDs only — fastest possible query
];
```

---

## Snippet 4 — Defer & Async Scripts

```php
add_filter( 'script_loader_tag', 'optimise_script_loading', 10, 3 );
function optimise_script_loading( string $tag, string $handle, string $src ): string {
    // These run after page load — async
    $async = [ 'google-analytics', 'facebook-pixel', 'hotjar' ];

    // These are non-critical but need DOM — defer
    $defer = [ 'my-chat-widget', 'cookie-notice', 'social-share' ];

    if ( in_array( $handle, $async, true ) ) {
        return str_replace( ' src=', ' async src=', $tag );
    }

    if ( in_array( $handle, $defer, true ) ) {
        return str_replace( ' src=', ' defer src=', $tag );
    }

    return $tag;
}
```

---

## Snippet 5 — Remove Unused Default Scripts & Styles

```php
add_action( 'wp_enqueue_scripts', 'remove_unused_assets', 100 );
function remove_unused_assets(): void {
    // Emoji (most sites don't need this)
    remove_action( 'wp_head', 'print_emoji_detection_script', 7 );
    remove_action( 'wp_print_styles', 'print_emoji_styles' );
    remove_action( 'admin_print_scripts', 'print_emoji_detection_script' );
    remove_action( 'admin_print_styles', 'print_emoji_styles' );

    // Block library CSS — remove if not using Gutenberg blocks on frontend
    wp_dequeue_style( 'wp-block-library' );
    wp_dequeue_style( 'wp-block-library-theme' );
    wp_dequeue_style( 'global-styles' );
    wp_dequeue_style( 'classic-theme-styles' );

    // jQuery migrate — only needed for very old code
    if ( ! is_admin() ) {
        wp_deregister_script( 'jquery' );
        wp_register_script( 'jquery', includes_url( '/js/jquery/jquery.min.js' ), [], '3.7.1', true );
    }
}
```

---

## Snippet 6 — Lazy Load Images

```php
// WordPress 5.5+ adds loading="lazy" by default on wp_get_attachment_image().
// For older setups or to ensure it everywhere:
add_filter( 'wp_get_attachment_image_attributes', function( array $attr ): array {
    $attr['loading'] = 'lazy';
    return $attr;
});

// Set fetchpriority="high" on the LCP (hero) image — never lazy-load this one
add_filter( 'wp_get_attachment_image_attributes', function( array $attr, WP_Post $attachment ): array {
    if ( is_singular() && (int) get_post_thumbnail_id() === $attachment->ID ) {
        $attr['fetchpriority'] = 'high';
        $attr['loading']       = 'eager';
    }
    return $attr;
}, 10, 2 );
```

---

## Snippet 7 — Preconnect to External Origins

```php
add_action( 'wp_head', 'preconnect_to_origins', 1 );
function preconnect_to_origins(): void {
    $origins = [
        'https://fonts.googleapis.com',
        'https://fonts.gstatic.com',
        'https://www.googletagmanager.com',
    ];

    foreach ( $origins as $origin ) {
        printf( '<link rel="preconnect" href="%s" crossorigin>' . "\n", esc_url( $origin ) );
    }
}
```

---

## Snippet 8 — Preload Critical Assets

```php
add_action( 'wp_head', 'preload_critical_assets', 1 );
function preload_critical_assets(): void {
    // Preload your main font
    printf(
        '<link rel="preload" href="%s" as="font" type="font/woff2" crossorigin>' . "\n",
        esc_url( get_template_directory_uri() . '/fonts/main.woff2' )
    );

    // Preload critical CSS
    printf(
        '<link rel="preload" href="%s" as="style">' . "\n",
        esc_url( get_template_directory_uri() . '/css/critical.css' )
    );
}
```

---

## Snippet 9 — Limit Post Revisions

```php
// Add to wp-config.php — BEFORE the "stop editing" comment

// Keep only last 3 revisions per post
define( 'WP_POST_REVISIONS', 3 );

// Disable revisions entirely (not recommended for production content sites)
// define( 'WP_POST_REVISIONS', false );
```

---

## Snippet 10 — Reduce Autosave Interval

```php
// Default autosave is every 60 seconds — reduce DB writes on busy sites
// Add to wp-config.php
define( 'AUTOSAVE_INTERVAL', 180 ); // Every 3 minutes instead of 1
```

---

## Snippet 11 — Slow Down the Heartbeat API

```php
// Default heartbeat ticks every 15 seconds — it's a frequent AJAX call
add_filter( 'heartbeat_settings', 'slow_down_heartbeat' );
function slow_down_heartbeat( array $settings ): array {
    $settings['interval'] = 60; // Tick every 60 seconds

    return $settings;
}

// Disable heartbeat on frontend entirely
add_action( 'init', function() {
    if ( ! is_admin() ) {
        wp_deregister_script( 'heartbeat' );
    }
});
```

---

## Snippet 12 — Use get_template_part() with Cache

```php
// Cache the output of a template part
function get_cached_template_part( string $slug, string $name = '', int $expiry = HOUR_IN_SECONDS ): void {
    $cache_key = "template_part_{$slug}_{$name}_" . get_the_ID();
    $output    = wp_cache_get( $cache_key, 'template_parts' );

    if ( false === $output ) {
        ob_start();
        get_template_part( $slug, $name );
        $output = ob_get_clean();
        wp_cache_set( $cache_key, $output, 'template_parts', $expiry );
    }

    echo $output; // phpcs:ignore WordPress.Security.EscapeOutput
}
```

---

## Snippet 13 — Remove Query Strings from Static Assets

```php
// Query strings (e.g. ?ver=6.4) can prevent CDN caching
add_filter( 'script_loader_src', 'remove_query_strings', 15 );
add_filter( 'style_loader_src', 'remove_query_strings', 15 );
function remove_query_strings( string $src ): string {
    if ( strpos( $src, '?ver=' ) ) {
        $src = remove_query_arg( 'ver', $src );
    }
    return $src;
}
```

---

## Snippet 14 — Enable GZIP via .htaccess

```apacheconf
# Add to .htaccess for Apache servers
<IfModule mod_deflate.c>
    AddOutputFilterByType DEFLATE text/html text/plain text/xml
    AddOutputFilterByType DEFLATE text/css text/javascript
    AddOutputFilterByType DEFLATE application/javascript application/x-javascript
    AddOutputFilterByType DEFLATE application/json application/xml
    AddOutputFilterByType DEFLATE image/svg+xml
</IfModule>
```

---

## Snippet 15 — Browser Caching via .htaccess

```apacheconf
# Add to .htaccess — tells browsers to cache static assets
<IfModule mod_expires.c>
    ExpiresActive On
    ExpiresByType image/jpg        "access plus 1 year"
    ExpiresByType image/jpeg       "access plus 1 year"
    ExpiresByType image/png        "access plus 1 year"
    ExpiresByType image/webp       "access plus 1 year"
    ExpiresByType image/gif        "access plus 1 year"
    ExpiresByType image/svg+xml    "access plus 1 year"
    ExpiresByType text/css         "access plus 1 month"
    ExpiresByType application/javascript "access plus 1 month"
    ExpiresByType application/x-javascript "access plus 1 month"
    ExpiresByType text/html        "access plus 0 seconds"
</IfModule>
```

---

## Snippet 16 — Disable oEmbed (if not using embeds)

```php
// oEmbed adds extra HTTP requests and a script on every page
add_action( 'init', 'disable_oembed' );
function disable_oembed(): void {
    // Remove oEmbed discovery links
    remove_action( 'wp_head', 'wp_oembed_add_discovery_links' );
    remove_action( 'wp_head', 'wp_oembed_add_host_js' );

    // Remove REST API oEmbed endpoint
    remove_action( 'rest_api_init', 'wp_oembed_register_route' );

    // Remove oEmbed-specific JavaScript from frontend
    wp_deregister_script( 'wp-embed' );
}
```

---

## Snippet 17 — Disable WooCommerce Scripts on Non-WooCommerce Pages

```php
// Only load WooCommerce assets where needed
add_action( 'wp_enqueue_scripts', 'dequeue_woocommerce_assets', 99 );
function dequeue_woocommerce_assets(): void {
    if ( function_exists( 'is_woocommerce' ) && ! is_woocommerce() && ! is_cart() && ! is_checkout() ) {
        wp_dequeue_style( 'woocommerce-general' );
        wp_dequeue_style( 'woocommerce-smallscreen' );
        wp_dequeue_style( 'woocommerce-layout' );
        wp_dequeue_script( 'woocommerce' );
        wp_dequeue_script( 'wc-cart-fragments' );
    }
}
```

---

## Snippet 18 — Batch Delete Transients

```php
// Clean up all expired transients from the database
function delete_all_transients( string $prefix = '' ): int {
    global $wpdb;

    if ( $prefix ) {
        $like    = $wpdb->esc_like( '_transient_' . $prefix ) . '%';
        $deleted = $wpdb->query(
            $wpdb->prepare( "DELETE FROM {$wpdb->options} WHERE option_name LIKE %s", $like )
        );
    } else {
        $deleted = $wpdb->query(
            "DELETE FROM {$wpdb->options}
             WHERE option_name LIKE '_transient_%'
             AND option_name NOT LIKE '_transient_timeout_%'"
        );
    }

    wp_cache_flush();
    return (int) $deleted;
}
```

---

## Snippet 19 — Measure Query Performance (Debug Only)

```php
// Add to functions.php temporarily — shows slow queries
// NEVER leave this on in production!
if ( defined( 'WP_DEBUG' ) && WP_DEBUG ) {
    add_action( 'shutdown', function() {
        global $wpdb;

        if ( ! defined( 'SAVEQUERIES' ) || ! SAVEQUERIES ) {
            return;
        }

        $slow_queries = array_filter(
            $wpdb->queries,
            fn( $q ) => $q[1] > 0.05 // Queries taking more than 50ms
        );

        if ( $slow_queries ) {
            echo '<pre style="background:#fff;padding:20px;font-size:12px;">';
            echo '<strong>Slow Queries (' . count( $slow_queries ) . '):</strong>' . PHP_EOL;
            foreach ( $slow_queries as $query ) {
                printf( '%.4fs — %s%s', $query[1], $query[0], PHP_EOL );
            }
            echo '</pre>';
        }
    });
}
```

---

## Snippet 20 — Use wp_remote_get() with Timeout & Caching

```php
function fetch_external_api( string $url ): array|WP_Error {
    $cache_key = 'api_' . md5( $url );
    $cached    = get_transient( $cache_key );

    if ( false !== $cached ) {
        return $cached;
    }

    $response = wp_remote_get( $url, [
        'timeout'    => 10,    // Don't hang for more than 10 seconds
        'user-agent' => 'MyPlugin/1.0 (+' . home_url() . ')',
    ]);

    if ( is_wp_error( $response ) ) {
        return $response;
    }

    if ( 200 !== wp_remote_retrieve_response_code( $response ) ) {
        return new WP_Error( 'bad_response', 'API returned non-200 status.' );
    }

    $data = json_decode( wp_remote_retrieve_body( $response ), true );
    set_transient( $cache_key, $data, 15 * MINUTE_IN_SECONDS );

    return $data;
}
```

---

## Snippet 21 — Inline Critical CSS

```php
// Inline a small critical CSS file directly in <head> to eliminate render-blocking
add_action( 'wp_head', 'inline_critical_css', 1 );
function inline_critical_css(): void {
    $critical_css_file = get_template_directory() . '/css/critical.css';

    if ( ! file_exists( $critical_css_file ) ) {
        return;
    }

    $css = file_get_contents( $critical_css_file ); // phpcs:ignore
    echo '<style id="critical-css">' . $css . '</style>' . "\n"; // phpcs:ignore
}
```

---

## Snippet 22 — Conditionally Load Google Fonts

```php
// Load Google Fonts only when needed — not on every page
add_action( 'wp_enqueue_scripts', 'conditionally_load_fonts' );
function conditionally_load_fonts(): void {
    // Only load on pages that use specific templates
    if ( is_singular( [ 'post', 'page' ] ) ) {
        wp_enqueue_style(
            'google-fonts',
            'https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap',
            [],
            null
        );
    }
}
```

---

## Snippet 23 — Clean wp_options Autoloaded Data

```php
// Too many autoloaded options slows down every page load
// Run this to find the biggest autoloaded options
function get_large_autoloaded_options( int $limit = 20 ): array {
    global $wpdb;

    return $wpdb->get_results(
        $wpdb->prepare(
            "SELECT option_name, LENGTH(option_value) as size
             FROM {$wpdb->options}
             WHERE autoload = 'yes'
             ORDER BY size DESC
             LIMIT %d",
            $limit
        ),
        ARRAY_A
    );
}

// Disable autoload for a specific option
function disable_option_autoload( string $option_name ): void {
    global $wpdb;
    $wpdb->update(
        $wpdb->options,
        [ 'autoload' => 'no' ],
        [ 'option_name' => $option_name ],
        [ '%s' ],
        [ '%s' ]
    );
}
```

---

## Snippet 24 — Image Size Optimisation

```php
// Register only the image sizes you actually use in your theme
add_action( 'after_setup_theme', 'register_theme_image_sizes' );
function register_theme_image_sizes(): void {
    // Remove default sizes you don't use (saves disk space on upload)
    remove_image_size( 'medium_large' ); // 768px wide — often unused

    // Add your own sizes
    add_image_size( 'hero',      1920, 600,  true ); // Hard crop
    add_image_size( 'card',      600,  400,  true ); // Hard crop
    add_image_size( 'thumbnail', 300,  300,  true ); // Hard crop
    add_image_size( 'wide',      1200, 9999, false ); // Proportional
}

// Use WebP when available (WordPress 5.8+)
add_filter( 'wp_upload_image_mime_transforms', function( array $transforms ): array {
    foreach ( $transforms as $mime => &$transform ) {
        $transform[] = 'image/webp';
    }
    return $transforms;
});
```

---

## Snippet 25 — Performance Checklist (Code Reference)

```php
/**
 * WordPress Performance Checklist
 *
 * SERVER & HOSTING
 * ✅ PHP 8.2+ (each major version is significantly faster)
 * ✅ OPcache enabled on server
 * ✅ Redis or Memcached for object cache
 * ✅ CDN configured for static assets
 * ✅ GZIP / Brotli compression enabled
 *
 * DATABASE
 * ✅ no_found_rows = true on non-paginated queries
 * ✅ update_post_meta_cache = false when meta not needed
 * ✅ update_post_term_cache = false when terms not needed
 * ✅ Expensive results cached with transients or object cache
 * ✅ Post revisions limited (WP_POST_REVISIONS = 3)
 * ✅ Autoloaded options under 1MB total
 *
 * ASSETS
 * ✅ Scripts loaded in footer (true as last arg in wp_enqueue_script)
 * ✅ Non-critical scripts deferred or async
 * ✅ Unused default styles removed (emoji, block library)
 * ✅ Google Fonts loaded with display=swap
 * ✅ No query strings on static assets (breaks CDN caching)
 *
 * IMAGES
 * ✅ All images have width + height attributes (prevents CLS)
 * ✅ Hero/LCP image has fetchpriority="high" loading="eager"
 * ✅ Below-fold images have loading="lazy"
 * ✅ Images served as WebP
 * ✅ Only image sizes you use are registered
 *
 * CORE
 * ✅ Heartbeat interval increased or disabled on frontend
 * ✅ oEmbed disabled if not using embeds
 * ✅ XML-RPC disabled if not needed
 * ✅ Browser caching headers set via .htaccess or server config
 */
```
