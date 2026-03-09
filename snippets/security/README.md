# 🔐 Security Snippets

> Essential WordPress security patterns every developer should know.

---

## Sanitization Quick Reference

| Input Type | Sanitize Function | Escape for Output |
|---|---|---|
| Plain text | `sanitize_text_field()` | `esc_html()` |
| HTML content | `wp_kses_post()` | `wp_kses_post()` |
| URL | `esc_url_raw()` | `esc_url()` |
| Email | `sanitize_email()` | `esc_html()` |
| Integer | `absint()` | `absint()` |
| Float | `floatval()` | `floatval()` |
| Filename | `sanitize_file_name()` | `esc_attr()` |
| HTML class | `sanitize_html_class()` | `esc_attr()` |
| Textarea | `sanitize_textarea_field()` | `esc_textarea()` |
| Array key | `sanitize_key()` | `esc_attr()` |
| CSS value | `sanitize_hex_color()` | `esc_attr()` |

---

## Nonce Verification — Complete Pattern

```php
// 1. Create nonce in form/page
function render_my_form(): void {
    ?>
    <form method="post" action="<?php echo esc_url( admin_url('admin-post.php') ); ?>">
        <?php wp_nonce_field( 'my_form_action', 'my_nonce' ); ?>
        <input type="hidden" name="action" value="my_form_submit">
        <input type="text" name="user_name" value="">
        <button type="submit">Submit</button>
    </form>
    <?php
}

// 2. Handle form submission
add_action( 'admin_post_my_form_submit', 'process_my_form' );
add_action( 'admin_post_nopriv_my_form_submit', 'process_my_form' ); // Allow non-logged-in users

function process_my_form(): void {
    // Verify nonce
    if ( ! isset( $_POST['my_nonce'] ) ||
         ! wp_verify_nonce( sanitize_key( $_POST['my_nonce'] ), 'my_form_action' ) ) {
        wp_die( esc_html__( 'Security verification failed.', 'my-plugin' ) );
    }

    // Sanitize input
    $user_name = sanitize_text_field( wp_unslash( $_POST['user_name'] ?? '' ) );

    // Process...
    wp_safe_redirect( add_query_arg( 'success', '1', wp_get_referer() ) );
    exit;
}
```

---

## Capability Checks

```php
// Common capability checks
current_user_can( 'manage_options' )   // Admin settings
current_user_can( 'edit_posts' )       // Editor+
current_user_can( 'publish_posts' )    // Author+
current_user_can( 'read' )             // Subscriber+
current_user_can( 'edit_post', $id )   // Can edit specific post

// Custom capability for admin page
add_action( 'admin_menu', 'add_my_menu_page' );
function add_my_menu_page(): void {
    add_menu_page(
        __( 'My Settings', 'my-plugin' ),
        __( 'My Settings', 'my-plugin' ),
        'manage_options', // Required capability
        'my-settings',
        'render_my_settings_page'
    );
}

function render_my_settings_page(): void {
    // Double-check capability even inside the callback
    if ( ! current_user_can( 'manage_options' ) ) {
        wp_die( esc_html__( 'You do not have permission to access this page.', 'my-plugin' ) );
    }
    // Render page...
}
```

---

## Protect REST Endpoints

```php
// Require authentication for sensitive endpoint
register_rest_route( 'my-plugin/v1', '/sensitive-data', [
    'methods'             => WP_REST_Server::READABLE,
    'callback'            => 'get_sensitive_data',
    'permission_callback' => function() {
        return current_user_can( 'edit_posts' );
    },
]);

// Require authentication AND nonce for write operations
register_rest_route( 'my-plugin/v1', '/update', [
    'methods'             => WP_REST_Server::CREATABLE,
    'callback'            => 'handle_update',
    'permission_callback' => function() {
        return current_user_can( 'manage_options' ) && wp_verify_nonce(
            sanitize_key( $_SERVER['HTTP_X_WP_NONCE'] ?? '' ),
            'wp_rest'
        );
    },
]);
```

---

## Prevent SQL Injection — Always Use $wpdb->prepare()

```php
global $wpdb;

// ❌ Never do this
$wpdb->get_results( "SELECT * FROM {$wpdb->posts} WHERE post_author = " . $_GET['author'] );

// ✅ Always do this
$author_id = absint( $_GET['author'] ?? 0 );
$wpdb->get_results(
    $wpdb->prepare(
        "SELECT ID, post_title FROM {$wpdb->posts} WHERE post_author = %d AND post_status = %s",
        $author_id,
        'publish'
    )
);

// Placeholders:
// %d = integer
// %s = string
// %f = float
// %i = identifier (table/column name) - WP 6.2+
```

---

## Limit Login Attempts (No Plugin Needed)

```php
add_filter( 'authenticate', 'limit_login_attempts', 30, 3 );
function limit_login_attempts( $user, string $username, string $password ) {
    if ( empty( $username ) ) {
        return $user;
    }

    $transient_key = 'login_attempts_' . md5( $username );
    $attempts      = (int) get_transient( $transient_key );

    if ( $attempts >= 5 ) {
        return new WP_Error(
            'too_many_attempts',
            __( 'Too many failed login attempts. Please try again in 15 minutes.', 'security' )
        );
    }

    if ( is_wp_error( $user ) ) {
        set_transient( $transient_key, $attempts + 1, 15 * MINUTE_IN_SECONDS );
    }

    return $user;
}
```

---

## Disable XML-RPC (if not needed)

```php
// Completely disable XML-RPC
add_filter( 'xmlrpc_enabled', '__return_false' );

// Remove X-Pingback header
add_filter( 'wp_headers', function( $headers ) {
    unset( $headers['X-Pingback'] );
    return $headers;
});
```
