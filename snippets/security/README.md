# 🔐 Security

> 25 essential WordPress security snippets. Sanitize inputs, escape outputs, verify nonces — every time.

---

## The Golden Rules

```
NEVER trust user input.
ALWAYS sanitize on the way IN.
ALWAYS escape on the way OUT.
ALWAYS verify nonces on form submissions.
ALWAYS check capabilities before doing anything privileged.
```

---

## Snippet 1 — Sanitization Quick Reference

```php
// Plain text (strips all HTML)
$name = sanitize_text_field( wp_unslash( $_POST['name'] ?? '' ) );

// Textarea (strips HTML, keeps line breaks)
$message = sanitize_textarea_field( wp_unslash( $_POST['message'] ?? '' ) );

// Email address
$email = sanitize_email( wp_unslash( $_POST['email'] ?? '' ) );

// URL
$url = esc_url_raw( wp_unslash( $_POST['url'] ?? '' ) );

// Integer
$id = absint( $_POST['id'] ?? 0 );

// Float
$price = (float) $_POST['price'];

// Slug / key
$key = sanitize_key( wp_unslash( $_POST['key'] ?? '' ) );

// File name
$filename = sanitize_file_name( wp_unslash( $_FILES['file']['name'] ?? '' ) );

// HTML (allow safe tags only — for editors)
$content = wp_kses_post( wp_unslash( $_POST['content'] ?? '' ) );

// Hex color
$color = sanitize_hex_color( $_POST['color'] ?? '' );
```

---

## Snippet 2 — Escaping Quick Reference

```php
// HTML context
echo esc_html( $title );

// HTML attribute context
echo '<input value="' . esc_attr( $value ) . '">';

// URL context (href, src)
echo '<a href="' . esc_url( $url ) . '">';

// JavaScript context
echo '<script>var x = ' . wp_json_encode( $data ) . ';</script>';

// CSS context
echo '<div style="color: ' . esc_attr( $color ) . '">';

// Textarea
echo '<textarea>' . esc_textarea( $content ) . '</textarea>';

// Translation strings with variables
printf( esc_html__( 'Hello, %s!', 'my-plugin' ), esc_html( $name ) );
```

---

## Snippet 3 — Nonce in a Form

```php
// Output nonce field inside your form
function render_my_form(): void {
    ?>
    <form method="post" action="<?php echo esc_url( admin_url( 'admin-post.php' ) ); ?>">
        <?php wp_nonce_field( 'my_form_action', 'my_nonce_field' ); ?>
        <input type="hidden" name="action" value="my_form_submit">
        <input type="text" name="username" placeholder="Username">
        <button type="submit"><?php esc_html_e( 'Submit', 'my-plugin' ); ?></button>
    </form>
    <?php
}

// Verify nonce when form is submitted
add_action( 'admin_post_my_form_submit', 'handle_my_form' );
add_action( 'admin_post_nopriv_my_form_submit', 'handle_my_form' );
function handle_my_form(): void {
    if ( ! isset( $_POST['my_nonce_field'] ) ||
         ! wp_verify_nonce( sanitize_key( $_POST['my_nonce_field'] ), 'my_form_action' ) ) {
        wp_die( esc_html__( 'Security check failed.', 'my-plugin' ) );
    }

    $username = sanitize_text_field( wp_unslash( $_POST['username'] ?? '' ) );
    // Process safely...

    wp_safe_redirect( add_query_arg( 'success', '1', wp_get_referer() ) );
    exit;
}
```

---

## Snippet 4 — Nonce in AJAX Request

```php
// PHP — verify nonce in AJAX handler
add_action( 'wp_ajax_my_ajax_action', 'handle_my_ajax' );
add_action( 'wp_ajax_nopriv_my_ajax_action', 'handle_my_ajax' );
function handle_my_ajax(): void {
    check_ajax_referer( 'my_ajax_nonce', 'nonce' ); // Dies if invalid

    $post_id = absint( $_POST['post_id'] ?? 0 );
    // Do work...
    wp_send_json_success( [ 'post_id' => $post_id ] );
}

// JS — send nonce with request
// First localize the nonce: wp_localize_script( 'my-script', 'MyData', [ 'nonce' => wp_create_nonce( 'my_ajax_nonce' ) ] );
```

```javascript
const formData = new FormData();
formData.append( 'action', 'my_ajax_action' );
formData.append( 'nonce', MyData.nonce );
formData.append( 'post_id', postId );

fetch( MyData.ajax_url, { method: 'POST', body: formData } )
    .then( r => r.json() )
    .then( data => console.log( data ) );
```

---

## Snippet 5 — Capability Checks Cheat Sheet

```php
// Check before ANY privileged action
if ( ! current_user_can( 'manage_options' ) ) {
    wp_die( esc_html__( 'You do not have permission.', 'my-plugin' ) );
}

// Common capabilities
current_user_can( 'manage_options' )      // Admin settings
current_user_can( 'edit_posts' )          // Editor, Author, Admin
current_user_can( 'publish_posts' )       // Author+
current_user_can( 'edit_others_posts' )   // Editor+
current_user_can( 'manage_categories' )   // Editor+
current_user_can( 'upload_files' )        // Author+
current_user_can( 'read' )                // All logged-in users
current_user_can( 'edit_post', $post_id ) // Can edit specific post
current_user_can( 'delete_post', $post_id ) // Can delete specific post
```

---

## Snippet 6 — Protect Direct File Access

```php
// Add to the TOP of every PHP file in your plugin or theme
if ( ! defined( 'ABSPATH' ) ) {
    exit; // Exit if accessed directly
}
```

---

## Snippet 7 — Always Use $wpdb->prepare() for Queries

```php
global $wpdb;

// ❌ NEVER — SQL injection risk
$wpdb->get_results( "SELECT * FROM {$wpdb->posts} WHERE ID = " . $_GET['id'] );

// ✅ ALWAYS — parameterized query
$id      = absint( $_GET['id'] ?? 0 );
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT ID, post_title, post_status FROM {$wpdb->posts} WHERE ID = %d AND post_status = %s",
        $id,
        'publish'
    )
);

// Placeholders:
// %d = integer
// %s = string
// %f = float
// %i = identifier (table/column name) — WordPress 6.2+
```

---

## Snippet 8 — Validate & Sanitize Settings API Fields

```php
// Sanitize callback for a settings field
add_settings_field( 'my_email', 'Email', 'render_email_field', 'my-settings', 'main' );
register_setting( 'my-settings', 'my_email', [
    'sanitize_callback' => 'sanitize_my_email_option',
    'default'           => '',
] );

function sanitize_my_email_option( $value ): string {
    $email = sanitize_email( $value );

    if ( ! is_email( $email ) ) {
        add_settings_error( 'my_email', 'invalid_email', __( 'Please enter a valid email.', 'my-plugin' ) );
        return get_option( 'my_email', '' ); // Return old value on failure
    }

    return $email;
}
```

---

## Snippet 9 — Restrict Admin Pages to Specific Roles

```php
add_action( 'admin_init', 'restrict_admin_to_admins' );
function restrict_admin_to_admins(): void {
    // Allow AJAX requests through
    if ( defined( 'DOING_AJAX' ) && DOING_AJAX ) {
        return;
    }

    // Redirect non-admins away from wp-admin
    if ( ! current_user_can( 'edit_posts' ) ) {
        wp_safe_redirect( home_url() );
        exit;
    }
}
```

---

## Snippet 10 — Disable File Editing in Admin

```php
// Add to wp-config.php — prevents editing theme/plugin files via admin
define( 'DISALLOW_FILE_EDIT', true );

// Also disable file modifications (plugin installs, updates) if needed
define( 'DISALLOW_FILE_MODS', true );
```

---

## Snippet 11 — Limit Login Attempts Without a Plugin

```php
add_filter( 'authenticate', 'limit_login_attempts', 30, 3 );
function limit_login_attempts( $user, string $username, string $password ) {
    if ( empty( $username ) ) {
        return $user;
    }

    $ip            = sanitize_text_field( $_SERVER['REMOTE_ADDR'] ?? '' );
    $transient_key = 'login_attempts_' . md5( $username . $ip );
    $attempts      = (int) get_transient( $transient_key );
    $max_attempts  = 5;

    if ( $attempts >= $max_attempts ) {
        return new WP_Error(
            'too_many_attempts',
            sprintf(
                __( 'Too many failed login attempts. Try again in %d minutes.', 'security' ),
                15
            )
        );
    }

    if ( is_wp_error( $user ) ) {
        set_transient( $transient_key, $attempts + 1, 15 * MINUTE_IN_SECONDS );
    } else {
        delete_transient( $transient_key ); // Reset on success
    }

    return $user;
}
```

---

## Snippet 12 — Disable XML-RPC

```php
// Disable XML-RPC completely
add_filter( 'xmlrpc_enabled', '__return_false' );

// Remove X-Pingback header
add_filter( 'wp_headers', function( array $headers ): array {
    unset( $headers['X-Pingback'] );
    return $headers;
});

// Remove RSD link from head
remove_action( 'wp_head', 'rsd_link' );
```

---

## Snippet 13 — Hide WordPress Login Errors

```php
// Default error tells attackers which part was wrong (username vs password)
// Replace with a generic message
add_filter( 'login_errors', function(): string {
    return esc_html__( 'Incorrect login details. Please try again.', 'security' );
});
```

---

## Snippet 14 — Protect REST API from Unauthorised Access

```php
// Require login for all REST API endpoints
add_filter( 'rest_authentication_errors', function( $result ) {
    if ( ! empty( $result ) ) {
        return $result;
    }

    if ( ! is_user_logged_in() ) {
        return new WP_Error(
            'rest_not_logged_in',
            __( 'You must be logged in to use the REST API.', 'security' ),
            [ 'status' => 401 ]
        );
    }

    return $result;
});

// Tip: Only apply to non-public routes using conditional logic
// if you still need public routes (e.g. for a headless frontend).
```

---

## Snippet 15 — Add Security Headers

```php
add_action( 'send_headers', 'add_security_headers' );
function add_security_headers(): void {
    header( 'X-Content-Type-Options: nosniff' );
    header( 'X-Frame-Options: SAMEORIGIN' );
    header( 'X-XSS-Protection: 1; mode=block' );
    header( 'Referrer-Policy: strict-origin-when-cross-origin' );
    header( 'Permissions-Policy: geolocation=(), microphone=(), camera=()' );
}
```

---

## Snippet 16 — Validate File Uploads

```php
function validate_uploaded_image( array $file ): array {
    // Check file size (max 2MB)
    if ( $file['size'] > 2 * MB_IN_BYTES ) {
        $file['error'] = __( 'File size must be under 2MB.', 'my-plugin' );
        return $file;
    }

    // Check MIME type — never trust $_FILES['type']
    $allowed_types = [ 'image/jpeg', 'image/png', 'image/webp', 'image/gif' ];
    $file_type     = wp_check_filetype( $file['name'] );

    if ( ! in_array( $file_type['type'], $allowed_types, true ) ) {
        $file['error'] = __( 'Only JPG, PNG, WebP and GIF images are allowed.', 'my-plugin' );
        return $file;
    }

    return $file;
}
add_filter( 'wp_handle_upload_prefilter', 'validate_uploaded_image' );
```

---

## Snippet 17 — Prevent Username Enumeration

```php
// Block ?author=1 style URL enumeration
add_action( 'init', 'prevent_author_enumeration' );
function prevent_author_enumeration(): void {
    if ( ! is_admin() && isset( $_GET['author'] ) ) {
        wp_safe_redirect( home_url(), 301 );
        exit;
    }
}

// Remove author from REST API user endpoint
add_filter( 'rest_endpoints', function( array $endpoints ): array {
    if ( ! is_user_logged_in() ) {
        unset( $endpoints['/wp/v2/users'] );
        unset( $endpoints['/wp/v2/users/(?P<id>[\d]+)'] );
    }
    return $endpoints;
});
```

---

## Snippet 18 — Disable Pingbacks

```php
// Disable self-pingbacks
add_action( 'pre_ping', 'disable_self_pingbacks' );
function disable_self_pingbacks( array &$links ): void {
    foreach ( $links as $l => $link ) {
        if ( str_starts_with( $link, home_url() ) ) {
            unset( $links[ $l ] );
        }
    }
}

// Disable all pingbacks via post type support
add_filter( 'xmlrpc_methods', function( array $methods ): array {
    unset( $methods['pingback.ping'] );
    return $methods;
});
```

---

## Snippet 19 — Content Security Policy Header

```php
add_action( 'send_headers', 'add_content_security_policy' );
function add_content_security_policy(): void {
    // Adjust sources to match your actual CDN/font/script sources
    $csp = implode( '; ', [
        "default-src 'self'",
        "script-src 'self' 'unsafe-inline' https://www.googletagmanager.com",
        "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
        "font-src 'self' https://fonts.gstatic.com",
        "img-src 'self' data: https:",
        "connect-src 'self'",
        "frame-ancestors 'none'",
    ]);

    header( "Content-Security-Policy: {$csp}" );
}
```

---

## Snippet 20 — Sanitize Arrays Recursively

```php
/**
 * Recursively sanitize an array of text values.
 *
 * @param array $data Raw input array.
 * @return array Sanitized array.
 */
function sanitize_array_recursive( array $data ): array {
    foreach ( $data as $key => $value ) {
        if ( is_array( $value ) ) {
            $data[ $key ] = sanitize_array_recursive( $value );
        } else {
            $data[ $key ] = sanitize_text_field( wp_unslash( (string) $value ) );
        }
    }
    return $data;
}

// Usage
$clean_data = sanitize_array_recursive( $_POST['my_array'] ?? [] );
```

---

## Snippet 21 — Force Strong Passwords

```php
add_action( 'user_profile_update_errors', 'enforce_strong_password', 10, 3 );
function enforce_strong_password( WP_Error $errors, bool $update, $user ): void {
    if ( isset( $user->user_pass ) && strlen( $user->user_pass ) < 12 ) {
        $errors->add(
            'weak_password',
            __( '<strong>Error:</strong> Password must be at least 12 characters long.', 'security' )
        );
    }
}
```

---

## Snippet 22 — Log Failed Login Attempts

```php
add_action( 'wp_login_failed', 'log_failed_login' );
function log_failed_login( string $username ): void {
    $ip        = sanitize_text_field( $_SERVER['REMOTE_ADDR'] ?? 'unknown' );
    $log_entry = sprintf(
        '[%s] Failed login for "%s" from IP %s' . PHP_EOL,
        current_time( 'Y-m-d H:i:s' ),
        sanitize_user( $username ),
        $ip
    );

    // Write to a private log file (outside webroot ideally)
    $log_file = WP_CONTENT_DIR . '/private/login-failures.log';
    file_put_contents( $log_file, $log_entry, FILE_APPEND | LOCK_EX );
}
```

---

## Snippet 23 — Disable Application Passwords (WordPress 5.6+)

```php
// If you don't use REST API authentication, disable application passwords
add_filter( 'wp_is_application_passwords_available', '__return_false' );
```

---

## Snippet 24 — Obscure wp-config.php & .htaccess via .htaccess

```apacheconf
# Add to your .htaccess to block direct access to sensitive files
<FilesMatch "^(wp-config\.php|\.htaccess|readme\.html|license\.txt|xmlrpc\.php)$">
    Order Allow,Deny
    Deny from all
</FilesMatch>

# Block directory browsing
Options -Indexes

# Block access to PHP files in uploads folder
<Directory "/wp-content/uploads">
    <FilesMatch "\.php$">
        Order Allow,Deny
        Deny from all
    </FilesMatch>
</Directory>
```

---

## Snippet 25 — Security Audit Checklist (Code Comments)

```php
/**
 * WordPress Security Checklist
 *
 * Before going live, verify each item:
 *
 * INPUT HANDLING
 * ✅ All $_POST/$_GET/$_REQUEST data is sanitized with sanitize_*()
 * ✅ wp_unslash() used before sanitizing
 * ✅ All database queries use $wpdb->prepare()
 * ✅ absint() used for all integer IDs
 *
 * OUTPUT HANDLING
 * ✅ All echoed strings use esc_html(), esc_attr(), esc_url()
 * ✅ No raw user data is printed directly
 *
 * AUTHENTICATION
 * ✅ Nonces verified on all form submissions (wp_verify_nonce)
 * ✅ Nonces verified on all AJAX requests (check_ajax_referer)
 * ✅ current_user_can() checked before every privileged action
 *
 * CONFIGURATION
 * ✅ ABSPATH check at top of every PHP file
 * ✅ DISALLOW_FILE_EDIT set in wp-config.php
 * ✅ Debug mode off on production (WP_DEBUG = false)
 * ✅ Database prefix changed from default wp_
 * ✅ Admin username is NOT "admin"
 *
 * SERVER
 * ✅ PHP files in /uploads blocked via .htaccess
 * ✅ Directory listing disabled
 * ✅ SSL certificate installed and enforced
 * ✅ wp-config.php protected
 */
```
