# 🚀 WordPress Developer Snippets & Cheat Sheet

> A curated collection of 225+ battle-tested WordPress code snippets, best practices, and cheat sheets for developers. Stop Googling the same things — everything you need in one place.

![GitHub Stars](https://img.shields.io/github/stars/ahmodmusa/wordpress-snippets?style=social)
![License](https://img.shields.io/badge/license-MIT-blue.svg)
![WordPress](https://img.shields.io/badge/WordPress-6.x-21759B?logo=wordpress)
![PHP](https://img.shields.io/badge/PHP-8.0%2B-777BB4?logo=php&logoColor=white)
![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)

---

## ⭐ Why Star This Repo?

- **225+ ready-to-use snippets** covering every aspect of WordPress development
- Organised by category — find what you need in seconds
- Covers modern WordPress: Gutenberg, REST API, Full Site Editing
- Each snippet is **tested, documented, and production-ready**
- Updated regularly with new snippets from the community

---

## 📚 Table of Contents

- [🪝 Hooks (Actions & Filters)](#-hooks-actions--filters)
- [🔍 WP\_Query & Database](#-wp_query--database)
- [🔐 Security](#-security)
- [⚡ Performance](#-performance)
- [🧱 Gutenberg & Block Editor](#-gutenberg--block-editor)
- [🔄 AJAX](#-ajax)
- [🌐 REST API](#-rest-api)
- [🎨 Theme Development](#-theme-development)
- [🔌 Plugin Development](#-plugin-development)
- [🤝 Contributing](#-contributing)

---

## 🪝 Hooks (Actions & Filters)

### Enqueue Scripts & Styles Correctly
```php
add_action( 'wp_enqueue_scripts', 'my_theme_scripts' );
function my_theme_scripts(): void {
    wp_enqueue_style(
        'my-style',
        get_template_directory_uri() . '/css/style.css',
        [],
        filemtime( get_template_directory() . '/css/style.css' )
    );

    wp_enqueue_script(
        'my-script',
        get_template_directory_uri() . '/js/main.js',
        [ 'jquery' ],
        filemtime( get_template_directory() . '/js/main.js' ),
        true // Load in footer
    );

    wp_localize_script( 'my-script', 'MyData', [
        'ajax_url' => admin_url( 'admin-ajax.php' ),
        'nonce'    => wp_create_nonce( 'my_nonce' ),
    ]);
}
```

### Remove Unnecessary Default Scripts & Styles
```php
add_action( 'wp_enqueue_scripts', 'remove_unnecessary_assets', 100 );
function remove_unnecessary_assets(): void {
    remove_action( 'wp_head', 'print_emoji_detection_script', 7 );
    remove_action( 'wp_print_styles', 'print_emoji_styles' );
    wp_dequeue_style( 'wp-block-library' );
    wp_dequeue_style( 'wp-block-library-theme' );
    wp_dequeue_style( 'global-styles' );
}
```

### Redirect After Login Based on Role
```php
add_filter( 'login_redirect', 'redirect_after_login', 10, 3 );
function redirect_after_login( string $redirect_to, string $requested, $user ): string {
    if ( isset( $user->roles ) && in_array( 'administrator', $user->roles, true ) ) {
        return admin_url();
    }
    return home_url( '/dashboard/' );
}
```

➡️ [See all 25 hook snippets](snippets/hooks/README.md)

---

## 🔍 WP\_Query & Database

### The Perfect WP_Query Template
```php
$args = [
    'post_type'      => 'post',
    'post_status'    => 'publish',
    'posts_per_page' => 10,
    'paged'          => get_query_var( 'paged', 1 ),
    'no_found_rows'  => false,
];

$query = new WP_Query( $args );

if ( $query->have_posts() ) :
    while ( $query->have_posts() ) : $query->the_post();
        the_title( '<h2>', '</h2>' );
    endwhile;
    wp_reset_postdata(); // ALWAYS reset after custom query
endif;
```

### Optimised Query — Skip What You Don't Need
```php
$args = [
    'post_type'              => 'post',
    'posts_per_page'         => 5,
    'no_found_rows'          => true,  // Skip COUNT(*) — no pagination needed
    'update_post_meta_cache' => false, // Skip if not using post meta
    'update_post_term_cache' => false, // Skip if not using terms
    'fields'                 => 'ids', // Return IDs only — fastest
];
```

### Safe Direct Database Query
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

➡️ [See all 25 query snippets](snippets/queries/README.md)

---

## 🔐 Security

### The Golden Rules
```php
// ❌ NEVER
$name = $_POST['name'];
echo $_GET['search'];
$wpdb->query( "SELECT * FROM wp_posts WHERE ID = " . $_GET['id'] );

// ✅ ALWAYS
$name    = sanitize_text_field( wp_unslash( $_POST['name'] ?? '' ) );
$search  = esc_html( sanitize_text_field( wp_unslash( $_GET['search'] ?? '' ) ) );
$results = $wpdb->get_results( $wpdb->prepare( "SELECT * FROM {$wpdb->posts} WHERE ID = %d", absint( $_GET['id'] ) ) );
```

### Verify Nonces on Form Submission
```php
// Output nonce in your form
wp_nonce_field( 'save_my_options', 'my_nonce_field' );

// Verify on processing
add_action( 'admin_post_save_my_options', 'handle_save_my_options' );
function handle_save_my_options(): void {
    if ( ! isset( $_POST['my_nonce_field'] ) ||
         ! wp_verify_nonce( sanitize_key( $_POST['my_nonce_field'] ), 'save_my_options' ) ) {
        wp_die( 'Security check failed.' );
    }
    if ( ! current_user_can( 'manage_options' ) ) {
        wp_die( 'Insufficient permissions.' );
    }
    // Process safely...
}
```

### Protect Direct File Access
```php
// Add to the top of every PHP file in your plugin or theme
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}
```

➡️ [See all 25 security snippets](snippets/security/README.md)

---

## ⚡ Performance

### Transients — Cache Expensive Operations
```php
function get_featured_posts(): array {
    $cache_key = 'featured_posts_v1';
    $cached    = get_transient( $cache_key );

    if ( false !== $cached ) {
        return $cached;
    }

    $posts = get_posts([ 'meta_key' => '_is_featured', 'meta_value' => '1', 'numberposts' => 6 ]);
    set_transient( $cache_key, $posts, HOUR_IN_SECONDS );

    return $posts;
}

add_action( 'save_post', fn() => delete_transient( 'featured_posts_v1' ) );
```

### Slow Down the Heartbeat API
```php
add_filter( 'heartbeat_settings', function( array $settings ): array {
    $settings['interval'] = 60; // Default is 15 seconds
    return $settings;
});
```

### Defer Non-Critical Scripts
```php
add_filter( 'script_loader_tag', 'defer_non_critical_scripts', 10, 3 );
function defer_non_critical_scripts( string $tag, string $handle ): string {
    $defer = [ 'my-chat-widget', 'cookie-notice' ];
    if ( in_array( $handle, $defer, true ) ) {
        return str_replace( ' src=', ' defer src=', $tag );
    }
    return $tag;
}
```

➡️ [See all 25 performance snippets](snippets/performance/README.md)

---

## 🧱 Gutenberg & Block Editor

### Register a Custom Block Category
```php
add_filter( 'block_categories_all', 'register_my_block_category', 10, 2 );
function register_my_block_category( array $categories ): array {
    return array_merge(
        [[ 'slug' => 'my-plugin', 'title' => __( 'My Plugin Blocks', 'my-plugin' ), 'icon' => 'layout' ]],
        $categories
    );
}
```

### Disable Block Editor for Specific Post Types
```php
add_filter( 'use_block_editor_for_post_type', 'disable_gutenberg_for_post_types', 10, 2 );
function disable_gutenberg_for_post_types( bool $use_editor, string $post_type ): bool {
    $classic_types = [ 'testimonial', 'faq', 'team_member' ];
    return in_array( $post_type, $classic_types, true ) ? false : $use_editor;
}
```

### Filter Block Output on the Frontend
```php
add_filter( 'render_block_core/paragraph', function( string $content ): string {
    return str_replace( '<p', '<p data-block="paragraph"', $content );
});
```

➡️ [See all 25 Gutenberg snippets](snippets/gutenberg/README.md)

---

## 🔄 AJAX

### Complete AJAX Setup (PHP + JS)
```php
add_action( 'wp_ajax_my_action',        'handle_my_ajax' );
add_action( 'wp_ajax_nopriv_my_action', 'handle_my_ajax' );

function handle_my_ajax(): void {
    check_ajax_referer( 'my_nonce_action', 'nonce' );
    $post_id = absint( $_POST['post_id'] ?? 0 );
    if ( ! $post_id ) {
        wp_send_json_error( [ 'message' => 'Invalid post ID.' ] );
    }
    wp_send_json_success( [ 'post_id' => $post_id ] );
}
```

```javascript
const formData = new FormData();
formData.append( 'action', 'my_action' );
formData.append( 'nonce',  MyData.nonce );
formData.append( 'post_id', postId );

const res  = await fetch( MyData.ajax_url, { method: 'POST', body: formData });
const data = await res.json();
if ( data.success ) console.log( data.data );
```

➡️ [See all 25 AJAX snippets](snippets/ajax/README.md)

---

## 🌐 REST API

### Register a Custom Endpoint
```php
add_action( 'rest_api_init', function() {
    register_rest_route( 'my-plugin/v1', '/posts/(?P<id>\d+)', [
        'methods'             => WP_REST_Server::READABLE,
        'callback'            => 'my_get_post',
        'permission_callback' => '__return_true',
        'args'                => [
            'id' => [ 'required' => true, 'sanitize_callback' => 'absint' ],
        ],
    ]);
});

function my_get_post( WP_REST_Request $request ): WP_REST_Response|WP_Error {
    $post = get_post( $request->get_param( 'id' ) );
    if ( ! $post || 'publish' !== $post->post_status ) {
        return new WP_Error( 'not_found', 'Post not found.', [ 'status' => 404 ] );
    }
    return rest_ensure_response([
        'id'    => $post->ID,
        'title' => get_the_title( $post ),
        'link'  => get_permalink( $post ),
    ]);
}
```

➡️ [See all 25 REST API snippets](snippets/rest-api/README.md)

---

## 🎨 Theme Development

### functions.php Boilerplate
```php
add_action( 'after_setup_theme', 'my_theme_setup' );
function my_theme_setup(): void {
    load_theme_textdomain( 'my-theme', get_template_directory() . '/languages' );
    add_theme_support( 'title-tag' );
    add_theme_support( 'post-thumbnails' );
    add_theme_support( 'html5', [ 'search-form', 'comment-form', 'comment-list', 'gallery', 'caption' ] );
    add_theme_support( 'wp-block-styles' );
    add_theme_support( 'align-wide' );
    register_nav_menus([
        'primary' => __( 'Primary Menu', 'my-theme' ),
        'footer'  => __( 'Footer Menu', 'my-theme' ),
    ]);
}
```

### Add Open Graph Meta Tags
```php
add_action( 'wp_head', 'output_open_graph_tags' );
function output_open_graph_tags(): void {
    if ( ! is_singular() ) return;
    echo '<meta property="og:title" content="'   . esc_attr( get_the_title() )   . '">' . "\n";
    echo '<meta property="og:url" content="'     . esc_url( get_permalink() )     . '">' . "\n";
    echo '<meta property="og:image" content="'   . esc_url( get_the_post_thumbnail_url( null, 'large' ) ) . '">' . "\n";
}
```

➡️ [See all 25 theme snippets](snippets/theme/README.md)

---

## 🔌 Plugin Development

### Plugin Header Boilerplate
```php
<?php
/**
 * Plugin Name:       My Awesome Plugin
 * Plugin URI:        https://github.com/ahmodmusa/my-plugin
 * Description:       A brief description of what this plugin does.
 * Version:           1.0.0
 * Requires at least: 6.0
 * Requires PHP:      8.0
 * Author:            Ahmod Musa
 * Author URI:        https://ahmodmusa.com
 * License:           GPL v2 or later
 * Text Domain:       my-plugin
 */
if ( ! defined( 'ABSPATH' ) ) exit;
```

### Activation & Deactivation Hooks
```php
register_activation_hook( __FILE__, 'my_plugin_activate' );
function my_plugin_activate(): void {
    add_option( 'my_plugin_version', '1.0.0' );
    flush_rewrite_rules();
}

register_deactivation_hook( __FILE__, 'my_plugin_deactivate' );
function my_plugin_deactivate(): void {
    wp_clear_scheduled_hook( 'my_plugin_cron_hook' );
    flush_rewrite_rules();
}
```

➡️ [See all 25 plugin snippets](snippets/plugin/README.md)

---

## 📊 Snippet Count

| Category | Snippets |
|---|---|
| [🪝 Hooks](snippets/hooks/README.md) | 25 |
| [🔍 WP_Query & Database](snippets/queries/README.md) | 25 |
| [🔐 Security](snippets/security/README.md) | 25 |
| [⚡ Performance](snippets/performance/README.md) | 25 |
| [🧱 Gutenberg](snippets/gutenberg/README.md) | 25 |
| [🔄 AJAX](snippets/ajax/README.md) | 25 |
| [🌐 REST API](snippets/rest-api/README.md) | 25 |
| [🎨 Theme Development](snippets/theme/README.md) | 25 |
| [🔌 Plugin Development](snippets/plugin/README.md) | 25 |
| **Total** | **225** |

---

## 🤝 Contributing

Contributions are what make this repo great! 🎉

1. **Fork** the repo
2. **Create** your branch: `git checkout -b snippet/my-snippet`
3. **Add** your snippet following the [contribution guidelines](.github/CONTRIBUTING.md)
4. **Commit**: `git commit -m 'Add: short description'`
5. **Push**: `git push origin snippet/my-snippet`
6. **Open a Pull Request**

Please read [CONTRIBUTING.md](.github/CONTRIBUTING.md) first.

---

## 📣 Spread the Word

If this saved you time:
- ⭐ **Star this repo** — it helps other developers find it!
- Share on Twitter/X with `#WordPress #WebDev #OpenSource`
- Post in WordPress Facebook groups or Slack channels

---

## 👨‍💻 About the Author

Built and maintained by **Ahmod Musa** — a WordPress developer passionate about clean code and helping the community.

- 🌐 Website: [ahmodmusa.com](https://ahmodmusa.com)
- 💼 Fiverr: [fiverr.com/s/bd665oX](https://www.fiverr.com/s/bd665oX)
- 🐙 GitHub: [github.com/ahmodmusa](https://github.com/ahmodmusa)

Need custom WordPress development? **[Hire me on Fiverr →](https://www.fiverr.com/s/bd665oX)**

---

## 📄 License

MIT © [Ahmod Musa](https://ahmodmusa.com)

---

<p align="center">
  Made with ❤️ for the WordPress community<br>
  <a href="https://github.com/ahmodmusa/wordpress-snippets/stargazers">⭐ Star this repo if it helped you!</a>
</p>
