# 🚀 WordPress Developer Snippets & Cheat Sheet

> A curated collection of battle-tested WordPress code snippets, best practices, and cheat sheets for developers. Save hours of Googling — everything you need in one place.

![GitHub Stars](https://img.shields.io/github/stars/YOUR_USERNAME/wordpress-snippets?style=social)
![License](https://img.shields.io/badge/license-MIT-blue.svg)
![WordPress](https://img.shields.io/badge/WordPress-6.x-21759B?logo=wordpress)
![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)

---

## ⭐ Why Star This Repo?

- **200+ ready-to-use snippets** covering every aspect of WordPress development
- Organized by category — find what you need in seconds
- Covers modern WordPress: Gutenberg, REST API, Full Site Editing
- Updated regularly with new snippets from the community
- Each snippet is **tested, documented, and production-ready**

---

## 📚 Table of Contents

- [🪝 Hooks (Actions & Filters)](#-hooks-actions--filters)
- [🔍 WP_Query & Database](#-wp_query--database)
- [🔐 Security](#-security)
- [⚡ Performance](#-performance)
- [🧱 Gutenberg & Block Editor](#-gutenberg--block-editor)
- [🔄 AJAX](#-ajax)
- [🌐 REST API](#-rest-api)
- [🎨 Theme Development](#-theme-development)
- [🔌 Plugin Development](#-plugin-development)
- [📋 Cheat Sheets](#-cheat-sheets)
- [🤝 Contributing](#-contributing)

---

## 🪝 Hooks (Actions & Filters)

### Add scripts & styles correctly
```php
add_action( 'wp_enqueue_scripts', 'my_theme_scripts' );
function my_theme_scripts() {
    // Enqueue style
    wp_enqueue_style(
        'my-style',
        get_template_directory_uri() . '/css/style.css',
        [],
        filemtime( get_template_directory() . '/css/style.css' )
    );

    // Enqueue script (deferred, dependent on jQuery)
    wp_enqueue_script(
        'my-script',
        get_template_directory_uri() . '/js/main.js',
        [ 'jquery' ],
        filemtime( get_template_directory() . '/js/main.js' ),
        true
    );

    // Pass PHP data to JS
    wp_localize_script( 'my-script', 'MyData', [
        'ajax_url' => admin_url( 'admin-ajax.php' ),
        'nonce'    => wp_create_nonce( 'my_nonce' ),
    ]);
}
```

### Remove default WordPress scripts/styles (speed boost)
```php
add_action( 'wp_enqueue_scripts', 'remove_unnecessary_scripts', 100 );
function remove_unnecessary_scripts() {
    // Remove emoji scripts
    remove_action( 'wp_head', 'print_emoji_detection_script', 7 );
    remove_action( 'wp_print_styles', 'print_emoji_styles' );

    // Remove block library CSS on frontend (if not using Gutenberg blocks)
    wp_dequeue_style( 'wp-block-library' );
    wp_dequeue_style( 'wp-block-library-theme' );
    wp_dequeue_style( 'global-styles' );
}
```

### Modify the login logo URL & title
```php
add_filter( 'login_headerurl', fn() => home_url() );
add_filter( 'login_headertext', fn() => get_bloginfo( 'name' ) );
```

### Add custom post status
```php
add_action( 'init', 'register_pending_review_status' );
function register_pending_review_status() {
    register_post_status( 'pending_review', [
        'label'                     => _x( 'Pending Review', 'post' ),
        'public'                    => false,
        'show_in_admin_all_list'    => true,
        'show_in_admin_status_list' => true,
        'label_count'               => _n_noop( 'Pending Review <span class="count">(%s)</span>', 'Pending Review <span class="count">(%s)</span>' ),
    ]);
}
```

➡️ [See all hook snippets](snippets/hooks/README.md)

---

## 🔍 WP_Query & Database

### The perfect WP_Query template
```php
$args = [
    'post_type'      => 'post',
    'post_status'    => 'publish',
    'posts_per_page' => 10,
    'paged'          => get_query_var( 'paged', 1 ),
    'orderby'        => 'date',
    'order'          => 'DESC',
    'meta_query'     => [
        [
            'key'     => '_featured',
            'value'   => '1',
            'compare' => '=',
        ],
    ],
    'tax_query' => [
        [
            'taxonomy' => 'category',
            'field'    => 'slug',
            'terms'    => [ 'news', 'updates' ],
        ],
    ],
];

$query = new WP_Query( $args );

if ( $query->have_posts() ) :
    while ( $query->have_posts() ) : $query->the_post();
        // Your loop code here
    endwhile;
    wp_reset_postdata(); // ALWAYS reset after custom query
endif;
```

### Get posts by meta value (clean version)
```php
function get_posts_by_meta( string $key, $value, string $post_type = 'post' ): array {
    return get_posts([
        'post_type'  => $post_type,
        'meta_key'   => $key,
        'meta_value' => $value,
        'numberposts' => -1,
    ]);
}
```

### Safe direct database query
```php
global $wpdb;

$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT post_id, meta_value FROM {$wpdb->postmeta} WHERE meta_key = %s AND meta_value > %d",
        'price',
        100
    ),
    ARRAY_A
);
```

➡️ [See all query snippets](snippets/queries/README.md)

---

## 🔐 Security

### Always sanitize, validate, escape — the golden rule
```php
// ❌ NEVER do this
$name = $_POST['name'];
echo $_GET['search'];
$wpdb->query( "SELECT * FROM wp_posts WHERE ID = " . $_GET['id'] );

// ✅ ALWAYS do this
$name   = sanitize_text_field( wp_unslash( $_POST['name'] ?? '' ) );
$search = esc_html( sanitize_text_field( wp_unslash( $_GET['search'] ?? '' ) ) );
$wpdb->get_results( $wpdb->prepare( "SELECT * FROM {$wpdb->posts} WHERE ID = %d", absint( $_GET['id'] ) ) );
```

### Verify nonces on form submission
```php
// Output nonce in your form
wp_nonce_field( 'save_my_options', 'my_nonce_field' );

// Verify on form processing
add_action( 'admin_post_save_my_options', 'handle_save_my_options' );
function handle_save_my_options() {
    if ( ! isset( $_POST['my_nonce_field'] ) ||
         ! wp_verify_nonce( sanitize_key( $_POST['my_nonce_field'] ), 'save_my_options' ) ) {
        wp_die( 'Security check failed.' );
    }

    // Check user capability
    if ( ! current_user_can( 'manage_options' ) ) {
        wp_die( 'Insufficient permissions.' );
    }

    // Process safely...
}
```

### Protect direct file access
```php
// Add to top of every plugin/theme PHP file
if ( ! defined( 'ABSPATH' ) ) {
    exit; // Exit if accessed directly
}
```

➡️ [See all security snippets](snippets/security/README.md)

---

## ⚡ Performance

### Transients — cache expensive operations
```php
function get_expensive_data(): array {
    $cache_key = 'my_expensive_data_v1';
    $cached    = get_transient( $cache_key );

    if ( false !== $cached ) {
        return $cached;
    }

    // Expensive operation (API call, complex query, etc.)
    $data = some_expensive_function();

    set_transient( $cache_key, $data, HOUR_IN_SECONDS );

    return $data;
}
```

### Object Cache helper
```php
function get_cached_posts( string $key, callable $callback, int $expiry = 3600 ): mixed {
    $cached = wp_cache_get( $key, 'my_plugin' );

    if ( false === $cached ) {
        $cached = $callback();
        wp_cache_set( $key, $cached, 'my_plugin', $expiry );
    }

    return $cached;
}
```

### Preload critical images (WordPress 6.3+)
```php
add_filter( 'wp_get_attachment_image_attributes', function( $attr, $attachment, $size ) {
    // fetchpriority for LCP image
    if ( is_singular() && has_post_thumbnail() && get_post_thumbnail_id() === $attachment->ID ) {
        $attr['fetchpriority'] = 'high';
        $attr['loading']       = 'eager';
    }
    return $attr;
}, 10, 3 );
```

➡️ [See all performance snippets](snippets/performance/README.md)

---

## 🧱 Gutenberg & Block Editor

### Register a custom block category
```php
add_filter( 'block_categories_all', function( $categories ) {
    return array_merge(
        [
            [
                'slug'  => 'my-plugin',
                'title' => __( 'My Plugin Blocks', 'my-plugin' ),
                'icon'  => 'wordpress',
            ],
        ],
        $categories
    );
});
```

### Enqueue block editor assets only in editor
```php
add_action( 'enqueue_block_editor_assets', 'my_block_editor_assets' );
function my_block_editor_assets() {
    wp_enqueue_script(
        'my-block-editor',
        plugins_url( 'build/index.js', __FILE__ ),
        [ 'wp-blocks', 'wp-element', 'wp-editor', 'wp-components' ],
        filemtime( plugin_dir_path( __FILE__ ) . 'build/index.js' )
    );
}
```

### Add custom block pattern
```php
add_action( 'init', 'register_my_block_patterns' );
function register_my_block_patterns() {
    register_block_pattern(
        'my-plugin/hero-section',
        [
            'title'       => __( 'Hero Section', 'my-plugin' ),
            'description' => __( 'A full-width hero with heading and CTA button', 'my-plugin' ),
            'categories'  => [ 'featured' ],
            'content'     => '<!-- wp:group {"align":"full"} --><div class="wp-block-group alignfull"><!-- wp:heading --><h2>Welcome</h2><!-- /wp:heading --></div><!-- /wp:group -->',
        ]
    );
}
```

➡️ [See all Gutenberg snippets](snippets/gutenberg/README.md)

---

## 🔄 AJAX

### Complete AJAX setup (PHP + JS)

**PHP (plugin/functions.php):**
```php
// Register AJAX action for logged-in users
add_action( 'wp_ajax_my_action', 'handle_my_ajax' );
// Register for non-logged-in users too
add_action( 'wp_ajax_nopriv_my_action', 'handle_my_ajax' );

function handle_my_ajax(): void {
    // Verify nonce
    check_ajax_referer( 'my_nonce', 'nonce' );

    // Get and sanitize data
    $post_id = absint( $_POST['post_id'] ?? 0 );

    if ( ! $post_id ) {
        wp_send_json_error( [ 'message' => 'Invalid post ID' ] );
    }

    // Do your work...
    $result = do_something( $post_id );

    wp_send_json_success( [ 'data' => $result ] );
}
```

**JavaScript:**
```javascript
async function sendAjax(postId) {
    const formData = new FormData();
    formData.append('action', 'my_action');
    formData.append('nonce', MyData.nonce);
    formData.append('post_id', postId);

    try {
        const response = await fetch(MyData.ajax_url, {
            method: 'POST',
            body: formData,
        });
        const data = await response.json();

        if (data.success) {
            console.log(data.data);
        } else {
            console.error(data.data.message);
        }
    } catch (error) {
        console.error('AJAX error:', error);
    }
}
```

➡️ [See all AJAX snippets](snippets/ajax/README.md)

---

## 🌐 REST API

### Register a custom REST endpoint
```php
add_action( 'rest_api_init', 'register_my_rest_routes' );
function register_my_rest_routes(): void {
    register_rest_route( 'my-plugin/v1', '/posts/(?P<id>\d+)', [
        'methods'             => WP_REST_Server::READABLE,
        'callback'            => 'my_get_post',
        'permission_callback' => '__return_true', // Public endpoint
        'args'                => [
            'id' => [
                'validate_callback' => fn( $v ) => is_numeric( $v ),
                'sanitize_callback' => 'absint',
                'required'          => true,
            ],
        ],
    ]);
}

function my_get_post( WP_REST_Request $request ): WP_REST_Response|WP_Error {
    $post = get_post( $request->get_param( 'id' ) );

    if ( ! $post ) {
        return new WP_Error( 'not_found', 'Post not found', [ 'status' => 404 ] );
    }

    return rest_ensure_response([
        'id'      => $post->ID,
        'title'   => get_the_title( $post ),
        'excerpt' => get_the_excerpt( $post ),
        'link'    => get_permalink( $post ),
    ]);
}
```

➡️ [See all REST API snippets](snippets/rest-api/README.md)

---

## 🎨 Theme Development

### theme.json starter (WordPress 6.x FSE)
```json
{
    "$schema": "https://schemas.wp.org/trunk/theme.json",
    "version": 2,
    "settings": {
        "color": {
            "palette": [
                { "slug": "primary", "color": "#0073aa", "name": "Primary" },
                { "slug": "secondary", "color": "#005177", "name": "Secondary" }
            ]
        },
        "typography": {
            "fontSizes": [
                { "slug": "small", "size": "14px", "name": "Small" },
                { "slug": "medium", "size": "18px", "name": "Medium" },
                { "slug": "large", "size": "24px", "name": "Large" }
            ]
        },
        "layout": {
            "contentSize": "800px",
            "wideSize": "1200px"
        }
    }
}
```

### functions.php boilerplate
```php
<?php
/**
 * Theme setup
 */
add_action( 'after_setup_theme', 'my_theme_setup' );
function my_theme_setup(): void {
    // Make theme available for translation
    load_theme_textdomain( 'my-theme', get_template_directory() . '/languages' );

    // Add theme supports
    add_theme_support( 'title-tag' );
    add_theme_support( 'post-thumbnails' );
    add_theme_support( 'html5', [ 'search-form', 'comment-form', 'comment-list', 'gallery', 'caption' ] );
    add_theme_support( 'customize-selective-refresh-widgets' );
    add_theme_support( 'wp-block-styles' );
    add_theme_support( 'align-wide' );

    // Register nav menus
    register_nav_menus([
        'primary' => __( 'Primary Menu', 'my-theme' ),
        'footer'  => __( 'Footer Menu', 'my-theme' ),
    ]);
}
```

➡️ [See all theme snippets](snippets/theme/README.md)

---

## 🔌 Plugin Development

### Plugin boilerplate header
```php
<?php
/**
 * Plugin Name:       My Awesome Plugin
 * Plugin URI:        https://github.com/YOUR_USERNAME/my-plugin
 * Description:       A brief description of what this plugin does.
 * Version:           1.0.0
 * Requires at least: 6.0
 * Requires PHP:      8.0
 * Author:            Your Name
 * Author URI:        https://yourwebsite.com
 * License:           GPL v2 or later
 * License URI:       https://www.gnu.org/licenses/gpl-2.0.html
 * Text Domain:       my-plugin
 * Domain Path:       /languages
 */

if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

define( 'MY_PLUGIN_VERSION', '1.0.0' );
define( 'MY_PLUGIN_PATH', plugin_dir_path( __FILE__ ) );
define( 'MY_PLUGIN_URL', plugin_dir_url( __FILE__ ) );
```

### Activation / Deactivation hooks
```php
register_activation_hook( __FILE__, 'my_plugin_activate' );
function my_plugin_activate(): void {
    // Create custom database table, set defaults, etc.
    add_option( 'my_plugin_version', MY_PLUGIN_VERSION );
    flush_rewrite_rules();
}

register_deactivation_hook( __FILE__, 'my_plugin_deactivate' );
function my_plugin_deactivate(): void {
    flush_rewrite_rules();
}
```

➡️ [See all plugin snippets](snippets/plugin/README.md)

---

## 📋 Cheat Sheets

### WordPress Conditional Tags
| Condition | Function |
|-----------|----------|
| Is front page? | `is_front_page()` |
| Is single post? | `is_single()` |
| Is page? | `is_page( $id_or_slug )` |
| Is archive? | `is_archive()` |
| Is category archive? | `is_category( $cat )` |
| Is tag archive? | `is_tag( $tag )` |
| Is search results? | `is_search()` |
| Is 404? | `is_404()` |
| User logged in? | `is_user_logged_in()` |
| Is admin? | `is_admin()` |
| In The Loop? | `in_the_loop()` |
| Has post thumbnail? | `has_post_thumbnail()` |

### Useful WordPress Constants
```php
ABSPATH          // Absolute path to WordPress root
WPINC            // Relative path to wp-includes
WP_CONTENT_DIR   // Absolute path to wp-content
WP_PLUGIN_DIR    // Absolute path to plugins
HOUR_IN_SECONDS  // 3600
DAY_IN_SECONDS   // 86400
WEEK_IN_SECONDS  // 604800
MONTH_IN_SECONDS // 2592000
YEAR_IN_SECONDS  // 31536000
```

### WP_Query Orderby options
```
date | modified | title | name | ID | rand | comment_count |
menu_order | meta_value | meta_value_num | post__in | relevance
```

---

## 🤝 Contributing

Contributions are what make this project amazing! 🎉

1. **Fork** the repo
2. **Create** your branch: `git checkout -b snippet/amazing-snippet`
3. **Add** your snippet following the [contribution guidelines](.github/blob/main/CONTRIBUTING.md)
4. **Commit**: `git commit -m 'Add: amazing snippet for X'`
5. **Push**: `git push origin snippet/amazing-snippet`
6. **Open a Pull Request**

Please read [CONTRIBUTING.md](.github/blob/main/CONTRIBUTING.md) before contributing.

---

## 📣 Spread the Word

If this repo saved you time, please:
- ⭐ **Star this repo** — it helps others find it!
- 🐦 Share it on Twitter/X with `#WordPress #WebDev`
- 📢 Post it in WordPress Facebook groups or Slack channels

---

## 📄 License

MIT © [Ahmod MUSA](https://github.com/ahmodmusa)

---

<p align="center">
  Made with ❤️ for the WordPress community<br>
  <a href="https://github.com/ahmodmusa/wordpress-snippets/stargazers">⭐ Star this repo if it helped you!</a>
</p>
