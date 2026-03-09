# 🔄 AJAX

> 25 snippets for handling AJAX requests in WordPress — the right way.

---

## How WordPress AJAX Works

```
Browser → POST to admin-ajax.php → WordPress loads → fires wp_ajax_{action} → sends JSON back
```

Every AJAX request needs:
1. An **action** name registered in PHP
2. A **nonce** for security
3. A **handler function** that ends with `wp_send_json_success()` or `wp_send_json_error()`

---

## Snippet 1 — Basic AJAX Setup (PHP + JS)

```php
// PHP — register the handler
add_action( 'wp_ajax_my_action',        'handle_my_ajax' ); // Logged-in users
add_action( 'wp_ajax_nopriv_my_action', 'handle_my_ajax' ); // Non-logged-in users

function handle_my_ajax(): void {
    check_ajax_referer( 'my_nonce_action', 'nonce' );

    $message = sanitize_text_field( wp_unslash( $_POST['message'] ?? '' ) );

    wp_send_json_success( [ 'echo' => $message ] );
}
```

```javascript
// JS — send the request
async function sendRequest( message ) {
    const formData = new FormData();
    formData.append( 'action', 'my_action' );
    formData.append( 'nonce',  MyData.nonce );
    formData.append( 'message', message );

    const response = await fetch( MyData.ajax_url, {
        method: 'POST',
        body:   formData,
    });

    const data = await response.json();

    if ( data.success ) {
        console.log( data.data ); // { echo: '...' }
    } else {
        console.error( data.data );
    }
}
```

---

## Snippet 2 — Localize AJAX URL and Nonce

```php
// Always localize before using AJAX in JS
add_action( 'wp_enqueue_scripts', 'localize_ajax_data' );
function localize_ajax_data(): void {
    wp_enqueue_script( 'my-script', get_template_directory_uri() . '/js/main.js', [], '1.0.0', true );

    wp_localize_script( 'my-script', 'MyAjax', [
        'url'   => admin_url( 'admin-ajax.php' ),
        'nonce' => wp_create_nonce( 'my_ajax_nonce' ),
    ]);
}
```

---

## Snippet 3 — AJAX for Logged-in Users Only

```php
// Only register wp_ajax_ (not wp_ajax_nopriv_) to restrict to logged-in users
add_action( 'wp_ajax_save_user_preference', 'save_user_preference' );

function save_user_preference(): void {
    check_ajax_referer( 'user_pref_nonce', 'nonce' );

    // current_user_can() works because user is logged in
    if ( ! current_user_can( 'read' ) ) {
        wp_send_json_error( [ 'message' => 'Insufficient permissions.' ], 403 );
    }

    $pref  = sanitize_key( wp_unslash( $_POST['pref'] ?? '' ) );
    $value = sanitize_text_field( wp_unslash( $_POST['value'] ?? '' ) );

    update_user_meta( get_current_user_id(), $pref, $value );

    wp_send_json_success( [ 'saved' => true ] );
}
```

---

## Snippet 4 — AJAX Load More Posts

```php
// PHP handler
add_action( 'wp_ajax_load_more_posts',        'ajax_load_more_posts' );
add_action( 'wp_ajax_nopriv_load_more_posts', 'ajax_load_more_posts' );

function ajax_load_more_posts(): void {
    check_ajax_referer( 'load_more_nonce', 'nonce' );

    $page      = absint( $_POST['page'] ?? 1 );
    $per_page  = 6;

    $query = new WP_Query([
        'post_type'      => 'post',
        'post_status'    => 'publish',
        'posts_per_page' => $per_page,
        'paged'          => $page,
        'no_found_rows'  => false,
    ]);

    if ( ! $query->have_posts() ) {
        wp_send_json_error( [ 'message' => 'No more posts.' ] );
    }

    $posts = [];
    while ( $query->have_posts() ) {
        $query->the_post();
        $posts[] = [
            'id'      => get_the_ID(),
            'title'   => get_the_title(),
            'excerpt' => get_the_excerpt(),
            'link'    => get_permalink(),
            'thumb'   => get_the_post_thumbnail_url( null, 'medium' ),
        ];
    }
    wp_reset_postdata();

    wp_send_json_success([
        'posts'    => $posts,
        'has_more' => $page < $query->max_num_pages,
    ]);
}
```

```javascript
// JS — Load More button handler
let currentPage = 1;

document.getElementById('load-more').addEventListener('click', async function() {
    this.textContent = 'Loading...';
    this.disabled = true;
    currentPage++;

    const formData = new FormData();
    formData.append( 'action', 'load_more_posts' );
    formData.append( 'nonce',  MyAjax.nonce );
    formData.append( 'page',   currentPage );

    const res  = await fetch( MyAjax.url, { method: 'POST', body: formData });
    const data = await res.json();

    if ( data.success ) {
        data.data.posts.forEach( post => {
            document.getElementById('posts-grid').insertAdjacentHTML('beforeend',
                `<article><h3><a href="${post.link}">${post.title}</a></h3><p>${post.excerpt}</p></article>`
            );
        });

        if ( ! data.data.has_more ) {
            this.style.display = 'none';
            return;
        }
    }

    this.textContent = 'Load More';
    this.disabled = false;
});
```

---

## Snippet 5 — AJAX Form Submission

```php
add_action( 'wp_ajax_submit_contact_form',        'handle_contact_form' );
add_action( 'wp_ajax_nopriv_submit_contact_form', 'handle_contact_form' );

function handle_contact_form(): void {
    check_ajax_referer( 'contact_form_nonce', 'nonce' );

    $name    = sanitize_text_field( wp_unslash( $_POST['name']    ?? '' ) );
    $email   = sanitize_email( wp_unslash( $_POST['email']        ?? '' ) );
    $message = sanitize_textarea_field( wp_unslash( $_POST['message'] ?? '' ) );

    // Validate
    $errors = [];
    if ( empty( $name ) )              $errors[] = 'Name is required.';
    if ( ! is_email( $email ) )        $errors[] = 'Valid email is required.';
    if ( strlen( $message ) < 10 )     $errors[] = 'Message is too short.';

    if ( $errors ) {
        wp_send_json_error( [ 'errors' => $errors ] );
    }

    // Send email
    $sent = wp_mail(
        get_option( 'admin_email' ),
        "Contact from {$name}",
        "Name: {$name}\nEmail: {$email}\n\nMessage:\n{$message}",
        [ "Reply-To: {$email}" ]
    );

    if ( $sent ) {
        wp_send_json_success( [ 'message' => 'Thank you! We will be in touch.' ] );
    } else {
        wp_send_json_error( [ 'message' => 'Failed to send. Please try again.' ] );
    }
}
```

---

## Snippet 6 — AJAX Search (Live Search)

```php
add_action( 'wp_ajax_live_search',        'handle_live_search' );
add_action( 'wp_ajax_nopriv_live_search', 'handle_live_search' );

function handle_live_search(): void {
    check_ajax_referer( 'live_search_nonce', 'nonce' );

    $term = sanitize_text_field( wp_unslash( $_POST['term'] ?? '' ) );

    if ( strlen( $term ) < 2 ) {
        wp_send_json_success( [ 'results' => [] ] );
    }

    $query = new WP_Query([
        's'              => $term,
        'post_type'      => [ 'post', 'page' ],
        'post_status'    => 'publish',
        'posts_per_page' => 5,
        'no_found_rows'  => true,
    ]);

    $results = [];
    foreach ( $query->posts as $post ) {
        $results[] = [
            'title' => get_the_title( $post ),
            'link'  => get_permalink( $post ),
            'type'  => $post->post_type,
        ];
    }

    wp_send_json_success( [ 'results' => $results ] );
}
```

```javascript
// JS — debounced live search
const searchInput = document.getElementById('search-input');
let debounceTimer;

searchInput.addEventListener('input', function() {
    clearTimeout( debounceTimer );
    debounceTimer = setTimeout( async () => {
        const term = this.value.trim();
        if ( term.length < 2 ) return;

        const formData = new FormData();
        formData.append( 'action', 'live_search' );
        formData.append( 'nonce',  MyAjax.nonce );
        formData.append( 'term',   term );

        const res  = await fetch( MyAjax.url, { method: 'POST', body: formData });
        const data = await res.json();

        if ( data.success ) {
            renderResults( data.data.results );
        }
    }, 300 ); // Wait 300ms after user stops typing
});
```

---

## Snippet 7 — AJAX Toggle Post Like / Favourite

```php
add_action( 'wp_ajax_toggle_like',        'handle_toggle_like' );
add_action( 'wp_ajax_nopriv_toggle_like', 'handle_toggle_like' );

function handle_toggle_like(): void {
    check_ajax_referer( 'like_nonce', 'nonce' );

    $post_id = absint( $_POST['post_id'] ?? 0 );

    if ( ! $post_id || ! get_post( $post_id ) ) {
        wp_send_json_error( [ 'message' => 'Invalid post.' ] );
    }

    $likes     = (int) get_post_meta( $post_id, '_like_count', true );
    $liked_key = 'liked_post_' . $post_id;

    // Use cookie for guests, user meta for logged-in users
    if ( is_user_logged_in() ) {
        $user_id = get_current_user_id();
        $liked   = (bool) get_user_meta( $user_id, $liked_key, true );

        if ( $liked ) {
            delete_user_meta( $user_id, $liked_key );
            $likes = max( 0, $likes - 1 );
        } else {
            update_user_meta( $user_id, $liked_key, '1' );
            $likes++;
        }
        $liked = ! $liked;
    } else {
        $liked = isset( $_COOKIE[ $liked_key ] );
        if ( $liked ) {
            setcookie( $liked_key, '', time() - 3600, '/' );
            $likes = max( 0, $likes - 1 );
        } else {
            setcookie( $liked_key, '1', time() + MONTH_IN_SECONDS, '/' );
            $likes++;
        }
        $liked = ! $liked;
    }

    update_post_meta( $post_id, '_like_count', $likes );

    wp_send_json_success( [ 'likes' => $likes, 'liked' => $liked ] );
}
```

---

## Snippet 8 — AJAX File Upload

```php
add_action( 'wp_ajax_upload_user_file', 'handle_user_file_upload' );

function handle_user_file_upload(): void {
    check_ajax_referer( 'upload_nonce', 'nonce' );

    if ( ! current_user_can( 'upload_files' ) ) {
        wp_send_json_error( [ 'message' => 'Permission denied.' ], 403 );
    }

    if ( empty( $_FILES['file'] ) ) {
        wp_send_json_error( [ 'message' => 'No file uploaded.' ] );
    }

    // Use WordPress's built-in upload handler
    require_once ABSPATH . 'wp-admin/includes/image.php';
    require_once ABSPATH . 'wp-admin/includes/file.php';
    require_once ABSPATH . 'wp-admin/includes/media.php';

    $attachment_id = media_handle_upload( 'file', 0 );

    if ( is_wp_error( $attachment_id ) ) {
        wp_send_json_error( [ 'message' => $attachment_id->get_error_message() ] );
    }

    wp_send_json_success([
        'attachment_id' => $attachment_id,
        'url'           => wp_get_attachment_url( $attachment_id ),
    ]);
}
```

---

## Snippet 9 — AJAX Delete Post

```php
add_action( 'wp_ajax_delete_my_post', 'handle_delete_post' );

function handle_delete_post(): void {
    check_ajax_referer( 'delete_post_nonce', 'nonce' );

    $post_id = absint( $_POST['post_id'] ?? 0 );

    if ( ! current_user_can( 'delete_post', $post_id ) ) {
        wp_send_json_error( [ 'message' => 'Permission denied.' ], 403 );
    }

    $deleted = wp_delete_post( $post_id, true ); // true = force delete (skip trash)

    if ( ! $deleted ) {
        wp_send_json_error( [ 'message' => 'Could not delete post.' ] );
    }

    wp_send_json_success( [ 'message' => 'Post deleted successfully.' ] );
}
```

---

## Snippet 10 — AJAX Save Post Meta

```php
add_action( 'wp_ajax_save_post_meta', 'handle_save_post_meta' );

function handle_save_post_meta(): void {
    check_ajax_referer( 'save_meta_nonce', 'nonce' );

    $post_id   = absint( $_POST['post_id'] ?? 0 );
    $meta_key  = sanitize_key( wp_unslash( $_POST['meta_key'] ?? '' ) );
    $meta_value = sanitize_text_field( wp_unslash( $_POST['meta_value'] ?? '' ) );

    if ( ! current_user_can( 'edit_post', $post_id ) ) {
        wp_send_json_error( [ 'message' => 'Permission denied.' ], 403 );
    }

    // Whitelist allowed meta keys
    $allowed_keys = [ '_subtitle', '_custom_color', '_show_sidebar' ];
    if ( ! in_array( $meta_key, $allowed_keys, true ) ) {
        wp_send_json_error( [ 'message' => 'Invalid meta key.' ] );
    }

    update_post_meta( $post_id, $meta_key, $meta_value );

    wp_send_json_success( [ 'saved' => true ] );
}
```

---

## Snippet 11 — Admin AJAX (Dashboard Actions)

```php
// Admin AJAX — only works when user is in wp-admin
add_action( 'wp_ajax_admin_bulk_action', 'handle_admin_bulk_action' );

function handle_admin_bulk_action(): void {
    check_ajax_referer( 'bulk_action_nonce', 'nonce' );

    if ( ! current_user_can( 'manage_options' ) ) {
        wp_send_json_error( [ 'message' => 'Admins only.' ], 403 );
    }

    $ids    = array_map( 'absint', $_POST['ids'] ?? [] );
    $action = sanitize_key( wp_unslash( $_POST['bulk_action'] ?? '' ) );

    $processed = 0;
    foreach ( $ids as $id ) {
        if ( 'publish' === $action ) {
            wp_update_post( [ 'ID' => $id, 'post_status' => 'publish' ] );
            $processed++;
        } elseif ( 'trash' === $action ) {
            wp_trash_post( $id );
            $processed++;
        }
    }

    wp_send_json_success( [ 'processed' => $processed ] );
}
```

---

## Snippet 12 — AJAX with wp_send_json_error Status Codes

```php
add_action( 'wp_ajax_my_action', 'handle_with_status_codes' );

function handle_with_status_codes(): void {
    check_ajax_referer( 'my_nonce', 'nonce' );

    $post_id = absint( $_POST['post_id'] ?? 0 );

    if ( ! $post_id ) {
        // 400 Bad Request
        wp_send_json_error( [ 'message' => 'post_id is required.' ], 400 );
    }

    if ( ! current_user_can( 'edit_post', $post_id ) ) {
        // 403 Forbidden
        wp_send_json_error( [ 'message' => 'Permission denied.' ], 403 );
    }

    if ( ! get_post( $post_id ) ) {
        // 404 Not Found
        wp_send_json_error( [ 'message' => 'Post not found.' ], 404 );
    }

    // 200 OK (default)
    wp_send_json_success( [ 'post_id' => $post_id ] );
}
```

---

## Snippet 13 — AJAX Pagination

```php
add_action( 'wp_ajax_paginate_posts',        'handle_paginate_posts' );
add_action( 'wp_ajax_nopriv_paginate_posts', 'handle_paginate_posts' );

function handle_paginate_posts(): void {
    check_ajax_referer( 'paginate_nonce', 'nonce' );

    $page     = absint( $_POST['page'] ?? 1 );
    $per_page = 9;
    $category = absint( $_POST['category'] ?? 0 );

    $args = [
        'post_type'      => 'post',
        'post_status'    => 'publish',
        'posts_per_page' => $per_page,
        'paged'          => $page,
    ];

    if ( $category ) {
        $args['cat'] = $category;
    }

    $query = new WP_Query( $args );

    ob_start();
    if ( $query->have_posts() ) :
        while ( $query->have_posts() ) : $query->the_post(); ?>
            <article id="post-<?php the_ID(); ?>" <?php post_class(); ?>>
                <h2><?php the_title(); ?></h2>
                <p><?php the_excerpt(); ?></p>
            </article>
        <?php endwhile;
        wp_reset_postdata();
    endif;
    $html = ob_get_clean();

    wp_send_json_success([
        'html'       => $html,
        'max_pages'  => $query->max_num_pages,
        'current'    => $page,
    ]);
}
```

---

## Snippet 14 — AJAX Rate Limiting with Transients

```php
add_action( 'wp_ajax_nopriv_rate_limited_action', 'handle_rate_limited_action' );

function handle_rate_limited_action(): void {
    check_ajax_referer( 'rate_limit_nonce', 'nonce' );

    $ip            = sanitize_text_field( $_SERVER['REMOTE_ADDR'] ?? '' );
    $transient_key = 'rate_limit_' . md5( $ip );
    $requests      = (int) get_transient( $transient_key );
    $max_requests  = 10; // Max 10 requests per minute

    if ( $requests >= $max_requests ) {
        wp_send_json_error( [ 'message' => 'Too many requests. Slow down.' ], 429 );
    }

    set_transient( $transient_key, $requests + 1, MINUTE_IN_SECONDS );

    // Process the request...
    wp_send_json_success( [ 'message' => 'Request processed.' ] );
}
```

---

## Snippet 15 — AJAX Return HTML Partial

```php
add_action( 'wp_ajax_get_post_card',        'ajax_get_post_card' );
add_action( 'wp_ajax_nopriv_get_post_card', 'ajax_get_post_card' );

function ajax_get_post_card(): void {
    check_ajax_referer( 'post_card_nonce', 'nonce' );

    $post_id = absint( $_POST['post_id'] ?? 0 );
    $post    = get_post( $post_id );

    if ( ! $post || 'publish' !== $post->post_status ) {
        wp_send_json_error( [ 'message' => 'Post not found.' ] );
    }

    ob_start();
    setup_postdata( $post );
    get_template_part( 'template-parts/card', 'post' );
    wp_reset_postdata();
    $html = ob_get_clean();

    wp_send_json_success( [ 'html' => $html ] );
}
```

---

## Snippet 16 — AJAX Filter Posts by Taxonomy

```php
add_action( 'wp_ajax_filter_by_category',        'handle_filter_by_category' );
add_action( 'wp_ajax_nopriv_filter_by_category', 'handle_filter_by_category' );

function handle_filter_by_category(): void {
    check_ajax_referer( 'filter_nonce', 'nonce' );

    $category_id = absint( $_POST['category_id'] ?? 0 );
    $page        = absint( $_POST['page'] ?? 1 );

    $args = [
        'post_type'      => 'post',
        'post_status'    => 'publish',
        'posts_per_page' => 9,
        'paged'          => $page,
    ];

    if ( $category_id ) {
        $args['tax_query'] = [[
            'taxonomy' => 'category',
            'field'    => 'term_id',
            'terms'    => $category_id,
        ]];
    }

    $query = new WP_Query( $args );
    $posts = [];

    while ( $query->have_posts() ) {
        $query->the_post();
        $posts[] = [
            'id'      => get_the_ID(),
            'title'   => get_the_title(),
            'link'    => get_permalink(),
            'excerpt' => wp_trim_words( get_the_excerpt(), 20 ),
            'thumb'   => get_the_post_thumbnail_url( null, 'card' ),
        ];
    }
    wp_reset_postdata();

    wp_send_json_success([
        'posts'     => $posts,
        'max_pages' => $query->max_num_pages,
    ]);
}
```

---

## Snippet 17 — AJAX Update Option (Settings Page)

```php
add_action( 'wp_ajax_save_plugin_option', 'handle_save_plugin_option' );

function handle_save_plugin_option(): void {
    check_ajax_referer( 'settings_nonce', 'nonce' );

    if ( ! current_user_can( 'manage_options' ) ) {
        wp_send_json_error( [ 'message' => 'Admins only.' ], 403 );
    }

    $key   = sanitize_key( wp_unslash( $_POST['option_key'] ?? '' ) );
    $value = sanitize_text_field( wp_unslash( $_POST['option_value'] ?? '' ) );

    // Whitelist which options can be saved
    $allowed_options = [ 'my_plugin_api_key', 'my_plugin_mode', 'my_plugin_color' ];
    if ( ! in_array( $key, $allowed_options, true ) ) {
        wp_send_json_error( [ 'message' => 'Option not allowed.' ] );
    }

    update_option( $key, $value );

    wp_send_json_success( [ 'message' => 'Option saved.' ] );
}
```

---

## Snippet 18 — AJAX Newsletter Subscribe

```php
add_action( 'wp_ajax_nopriv_newsletter_subscribe', 'handle_newsletter_subscribe' );
add_action( 'wp_ajax_newsletter_subscribe',        'handle_newsletter_subscribe' );

function handle_newsletter_subscribe(): void {
    check_ajax_referer( 'subscribe_nonce', 'nonce' );

    $email = sanitize_email( wp_unslash( $_POST['email'] ?? '' ) );

    if ( ! is_email( $email ) ) {
        wp_send_json_error( [ 'message' => 'Please enter a valid email address.' ] );
    }

    // Check if already subscribed
    $subscribers = get_option( 'my_newsletter_subscribers', [] );
    if ( in_array( $email, $subscribers, true ) ) {
        wp_send_json_error( [ 'message' => 'You are already subscribed.' ] );
    }

    // Add to list
    $subscribers[] = $email;
    update_option( 'my_newsletter_subscribers', $subscribers );

    // Notify admin
    wp_mail(
        get_option( 'admin_email' ),
        'New Newsletter Subscriber',
        "New subscriber: {$email}"
    );

    wp_send_json_success( [ 'message' => 'Thank you for subscribing!' ] );
}
```

---

## Snippet 19 — AJAX Autocomplete (jQuery UI / Custom)

```php
add_action( 'wp_ajax_autocomplete_posts',        'handle_autocomplete' );
add_action( 'wp_ajax_nopriv_autocomplete_posts', 'handle_autocomplete' );

function handle_autocomplete(): void {
    check_ajax_referer( 'autocomplete_nonce', 'nonce' );

    $term = sanitize_text_field( wp_unslash( $_GET['term'] ?? '' ) );

    if ( strlen( $term ) < 2 ) {
        wp_send_json_success( [] );
    }

    $posts = get_posts([
        's'              => $term,
        'post_type'      => 'post',
        'numberposts'    => 10,
        'post_status'    => 'publish',
        'no_found_rows'  => true,
    ]);

    $results = array_map( fn( $p ) => [
        'label' => get_the_title( $p ),
        'value' => get_permalink( $p ),
        'id'    => $p->ID,
    ], $posts );

    wp_send_json_success( $results );
}
```

---

## Snippet 20 — AJAX Check Username Availability

```php
add_action( 'wp_ajax_nopriv_check_username', 'check_username_availability' );

function check_username_availability(): void {
    check_ajax_referer( 'registration_nonce', 'nonce' );

    $username = sanitize_user( wp_unslash( $_POST['username'] ?? '' ) );

    if ( empty( $username ) ) {
        wp_send_json_error( [ 'message' => 'Username is required.' ] );
    }

    if ( username_exists( $username ) ) {
        wp_send_json_error( [ 'message' => 'Username is already taken.' ] );
    }

    if ( ! validate_username( $username ) ) {
        wp_send_json_error( [ 'message' => 'Username contains invalid characters.' ] );
    }

    wp_send_json_success( [ 'message' => 'Username is available!' ] );
}
```

---

## Snippet 21 — AJAX Refresh a Nonce

```php
// Nonces expire after 12-24hrs — refresh them for long-running pages
add_action( 'wp_ajax_refresh_nonce',        'refresh_nonce' );
add_action( 'wp_ajax_nopriv_refresh_nonce', 'refresh_nonce' );

function refresh_nonce(): void {
    $action = sanitize_key( wp_unslash( $_POST['nonce_action'] ?? '' ) );

    if ( empty( $action ) ) {
        wp_send_json_error( [ 'message' => 'No action provided.' ] );
    }

    wp_send_json_success( [ 'nonce' => wp_create_nonce( $action ) ] );
}
```

---

## Snippet 22 — Heartbeat API (Real-Time Data)

```php
// Piggyback on the Heartbeat API for real-time-ish updates
add_filter( 'heartbeat_received', 'handle_heartbeat_data', 10, 2 );
function handle_heartbeat_data( array $response, array $data ): array {
    if ( isset( $data['check_notifications'] ) ) {
        $user_id       = get_current_user_id();
        $notifications = get_user_meta( $user_id, '_unread_notifications', true );

        $response['notification_count'] = is_array( $notifications ) ? count( $notifications ) : 0;
    }

    return $response;
}
```

```javascript
// JS — send data with heartbeat
jQuery( document ).on( 'heartbeat-send', function( e, data ) {
    data['check_notifications'] = true;
});

jQuery( document ).on( 'heartbeat-tick', function( e, data ) {
    if ( data['notification_count'] !== undefined ) {
        document.getElementById('notification-badge').textContent = data['notification_count'];
    }
});
```

---

## Snippet 23 — AJAX Error Handling (JS)

```javascript
// Reusable AJAX helper with full error handling
async function wpAjax( action, data = {}, method = 'POST' ) {
    const formData = new FormData();
    formData.append( 'action', action );
    formData.append( 'nonce',  MyAjax.nonce );

    Object.entries( data ).forEach( ( [ key, value ] ) => {
        formData.append( key, value );
    });

    try {
        const response = await fetch( MyAjax.url, { method, body: formData });

        if ( ! response.ok ) {
            throw new Error( `HTTP error: ${response.status}` );
        }

        const json = await response.json();

        if ( ! json.success ) {
            throw new Error( json.data?.message || 'Unknown error.' );
        }

        return json.data;

    } catch ( error ) {
        console.error( `AJAX error [${action}]:`, error.message );
        throw error; // Re-throw so callers can handle it
    }
}

// Usage
try {
    const result = await wpAjax( 'load_more_posts', { page: 2 } );
    console.log( result.posts );
} catch ( e ) {
    showErrorMessage( e.message );
}
```

---

## Snippet 24 — AJAX with AbortController (Cancel Pending Requests)

```javascript
// Cancel previous request if user triggers a new one (e.g. live search)
let abortController = null;

async function searchPosts( term ) {
    // Cancel previous request
    if ( abortController ) {
        abortController.abort();
    }
    abortController = new AbortController();

    const formData = new FormData();
    formData.append( 'action', 'live_search' );
    formData.append( 'nonce',  MyAjax.nonce );
    formData.append( 'term',   term );

    try {
        const response = await fetch( MyAjax.url, {
            method: 'POST',
            body:   formData,
            signal: abortController.signal, // Pass abort signal
        });

        const data = await response.json();
        return data.success ? data.data.results : [];

    } catch ( error ) {
        if ( error.name === 'AbortError' ) {
            return []; // Request was cancelled — not a real error
        }
        throw error;
    }
}
```

---

## Snippet 25 — AJAX Security Checklist

```php
/**
 * WordPress AJAX Security Checklist
 *
 * Every AJAX handler must follow this pattern:
 *
 * ✅ 1. REGISTER the action correctly
 *    add_action( 'wp_ajax_{action}', ... )          — logged-in users
 *    add_action( 'wp_ajax_nopriv_{action}', ... )   — also allow guests (if needed)
 *
 * ✅ 2. VERIFY the nonce (always first line in handler)
 *    check_ajax_referer( 'my_nonce_action', 'nonce' );
 *    // OR: if ( ! wp_verify_nonce( ... ) ) { wp_send_json_error(...); }
 *
 * ✅ 3. CHECK capabilities
 *    if ( ! current_user_can( 'edit_posts' ) ) { wp_send_json_error(..., 403); }
 *
 * ✅ 4. SANITIZE every input
 *    sanitize_text_field(), absint(), sanitize_email(), sanitize_key(), etc.
 *
 * ✅ 5. VALIDATE the data makes sense
 *    Check required fields, validate formats, check records exist
 *
 * ✅ 6. DO the work
 *
 * ✅ 7. RESPOND with wp_send_json_success() or wp_send_json_error()
 *    Never use echo + die() — always use these functions
 *    They call wp_die() automatically
 *
 * ✅ 8. Never exit() or die() manually — wp_send_json_* handles this
 *
 * Complete handler template:
 */
add_action( 'wp_ajax_my_secure_action', 'my_secure_ajax_handler' );
function my_secure_ajax_handler(): void {
    // 1. Verify nonce
    check_ajax_referer( 'my_nonce_action', 'nonce' );

    // 2. Check capability
    if ( ! current_user_can( 'edit_posts' ) ) {
        wp_send_json_error( [ 'message' => 'Permission denied.' ], 403 );
    }

    // 3. Sanitize inputs
    $post_id = absint( $_POST['post_id'] ?? 0 );

    // 4. Validate
    if ( ! $post_id ) {
        wp_send_json_error( [ 'message' => 'Invalid post ID.' ], 400 );
    }

    // 5. Do work
    $result = do_something( $post_id );

    // 6. Respond
    wp_send_json_success( [ 'result' => $result ] );
}
```
