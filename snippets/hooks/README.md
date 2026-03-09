# 🪝 Hooks — Actions & Filters

> Complete guide to WordPress hooks with 25 battle-tested snippets.

---

## Action vs Filter — Quick Difference

| | Action | Filter |
|---|---|---|
| Purpose | Do something | Modify something |
| Return value | Not required | **Required** |
| Function | `add_action()` | `add_filter()` |
| Trigger | `do_action()` | `apply_filters()` |

---

## Hook Load Order (Page Request)

```
muplugins_loaded → plugins_loaded → setup_theme → after_setup_theme
→ init → wp_loaded → template_redirect → wp_head → [content] → wp_footer
```

---

## Snippet 1 — Enqueue Scripts & Styles Correctly

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
}
```

---

## Snippet 2 — Pass PHP Data to JavaScript

```php
add_action( 'wp_enqueue_scripts', 'my_localize_script' );
function my_localize_script(): void {
    wp_enqueue_script( 'my-script', get_template_directory_uri() . '/js/main.js', [], '1.0.0', true );

    wp_localize_script( 'my-script', 'MyData', [
        'ajax_url'  => admin_url( 'admin-ajax.php' ),
        'nonce'     => wp_create_nonce( 'my_nonce' ),
        'is_logged' => is_user_logged_in() ? 'yes' : 'no',
        'home_url'  => home_url(),
    ]);
}
```

---

## Snippet 3 — Enqueue Scripts Only on Specific Page

```php
add_action( 'wp_enqueue_scripts', 'load_scripts_on_contact_page' );
function load_scripts_on_contact_page(): void {
    if ( ! is_page( 'contact' ) ) {
        return;
    }

    wp_enqueue_script(
        'contact-form-script',
        get_template_directory_uri() . '/js/contact.js',
        [],
        '1.0.0',
        true
    );
}
```

---

## Snippet 4 — Enqueue Admin Scripts on Specific Admin Page

```php
add_action( 'admin_enqueue_scripts', 'my_admin_scripts' );
function my_admin_scripts( string $hook ): void {
    // Only load on your plugin's settings page
    if ( 'settings_page_my-plugin' !== $hook ) {
        return;
    }

    wp_enqueue_style( 'my-admin-style', plugin_dir_url( __FILE__ ) . 'css/admin.css' );
    wp_enqueue_script( 'my-admin-script', plugin_dir_url( __FILE__ ) . 'js/admin.js', [ 'jquery' ], '1.0.0', true );
}
```

---

## Snippet 5 — Remove Unnecessary Default Scripts & Styles

```php
add_action( 'wp_enqueue_scripts', 'remove_unnecessary_assets', 100 );
function remove_unnecessary_assets(): void {
    // Remove emoji scripts & styles
    remove_action( 'wp_head', 'print_emoji_detection_script', 7 );
    remove_action( 'wp_print_styles', 'print_emoji_styles' );

    // Remove block library CSS (if not using Gutenberg on frontend)
    wp_dequeue_style( 'wp-block-library' );
    wp_dequeue_style( 'wp-block-library-theme' );
    wp_dequeue_style( 'global-styles' );

    // Remove classic theme styles
    wp_dequeue_style( 'classic-theme-styles' );
}
```

---

## Snippet 6 — Remove WordPress Version from Head

```php
remove_action( 'wp_head', 'wp_generator' );

// Also remove from RSS feeds
add_filter( 'the_generator', '__return_empty_string' );
```

---

## Snippet 7 — Add Body Class Conditionally

```php
add_filter( 'body_class', 'custom_body_classes' );
function custom_body_classes( array $classes ): array {
    // Add class on all singular posts
    if ( is_singular( 'post' ) ) {
        $classes[] = 'single-blog-post';
    }

    // Add class when user is logged in
    if ( is_user_logged_in() ) {
        $classes[] = 'user-logged-in';
    }

    // Add class based on custom field
    if ( get_post_meta( get_the_ID(), 'dark_mode', true ) ) {
        $classes[] = 'dark-mode';
    }

    return $classes;
}
```

---

## Snippet 8 — Modify the Excerpt Length & More Tag

```php
// Change excerpt length (default: 55 words)
add_filter( 'excerpt_length', fn() => 30 );

// Change the "..." to a custom Read More link
add_filter( 'excerpt_more', function( $more ) {
    return ' <a class="read-more" href="' . esc_url( get_permalink() ) . '">'
        . __( 'Read More', 'my-theme' ) . '</a>';
});
```

---

## Snippet 9 — Change the Login Logo & URL

```php
// Change logo image
add_action( 'login_enqueue_scripts', 'custom_login_logo' );
function custom_login_logo(): void {
    ?>
    <style>
        #login h1 a {
            background-image: url('<?php echo esc_url( get_template_directory_uri() . '/img/logo.png' ); ?>');
            background-size: contain;
            width: 200px;
            height: 80px;
        }
    </style>
    <?php
}

// Change logo link URL and title
add_filter( 'login_headerurl', fn() => home_url() );
add_filter( 'login_headertext', fn() => get_bloginfo( 'name' ) );
```

---

## Snippet 10 — Add Custom Dashboard Widget

```php
add_action( 'wp_dashboard_setup', 'add_my_dashboard_widget' );
function add_my_dashboard_widget(): void {
    wp_add_dashboard_widget(
        'my_dashboard_widget',
        __( 'Quick Links', 'my-plugin' ),
        'render_my_dashboard_widget'
    );
}

function render_my_dashboard_widget(): void {
    echo '<ul>';
    echo '<li><a href="' . esc_url( admin_url( 'edit.php' ) ) . '">' . esc_html__( 'All Posts', 'my-plugin' ) . '</a></li>';
    echo '<li><a href="' . esc_url( home_url() ) . '" target="_blank">' . esc_html__( 'Visit Site', 'my-plugin' ) . '</a></li>';
    echo '</ul>';
}
```

---

## Snippet 11 — Hook Into Save Post

```php
add_action( 'save_post', 'on_post_save', 10, 3 );
function on_post_save( int $post_id, WP_Post $post, bool $update ): void {
    // Ignore autosaves
    if ( defined( 'DOING_AUTOSAVE' ) && DOING_AUTOSAVE ) {
        return;
    }

    // Only run for specific post type
    if ( 'post' !== $post->post_type ) {
        return;
    }

    // Check user capability
    if ( ! current_user_can( 'edit_post', $post_id ) ) {
        return;
    }

    // Your logic here — e.g. clear a cache
    delete_transient( 'latest_posts_cache' );
}
```

---

## Snippet 12 — Run Code on User Registration

```php
add_action( 'user_register', 'on_user_registered' );
function on_user_registered( int $user_id ): void {
    // Add default meta for new users
    update_user_meta( $user_id, 'onboarding_complete', '0' );

    // Send a custom welcome email
    $user = get_userdata( $user_id );
    wp_mail(
        $user->user_email,
        __( 'Welcome!', 'my-plugin' ),
        sprintf( __( 'Hi %s, thanks for registering!', 'my-plugin' ), $user->display_name )
    );
}
```

---

## Snippet 13 — Redirect After Login Based on Role

```php
add_filter( 'login_redirect', 'redirect_after_login', 10, 3 );
function redirect_after_login( string $redirect_to, string $requested_redirect_to, $user ): string {
    if ( isset( $user->roles ) && is_array( $user->roles ) ) {
        if ( in_array( 'administrator', $user->roles, true ) ) {
            return admin_url();
        }

        if ( in_array( 'editor', $user->roles, true ) ) {
            return admin_url( 'edit.php' );
        }

        // All other roles go to homepage
        return home_url( '/dashboard/' );
    }

    return $redirect_to;
}
```

---

## Snippet 14 — Add Custom Post Type Columns

```php
// Add the column header
add_filter( 'manage_book_posts_columns', 'add_book_columns' );
function add_book_columns( array $columns ): array {
    $columns['genre']  = __( 'Genre', 'my-plugin' );
    $columns['rating'] = __( 'Rating', 'my-plugin' );
    return $columns;
}

// Populate the column data
add_action( 'manage_book_posts_custom_column', 'render_book_columns', 10, 2 );
function render_book_columns( string $column, int $post_id ): void {
    if ( 'genre' === $column ) {
        $terms = get_the_terms( $post_id, 'genre' );
        echo $terms ? esc_html( implode( ', ', wp_list_pluck( $terms, 'name' ) ) ) : '—';
    }

    if ( 'rating' === $column ) {
        echo esc_html( get_post_meta( $post_id, '_rating', true ) ?: '—' );
    }
}
```

---

## Snippet 15 — Make Custom Column Sortable

```php
add_filter( 'manage_edit-book_sortable_columns', 'make_book_columns_sortable' );
function make_book_columns_sortable( array $columns ): array {
    $columns['rating'] = 'rating';
    return $columns;
}

add_action( 'pre_get_posts', 'sort_by_rating' );
function sort_by_rating( WP_Query $query ): void {
    if ( ! is_admin() || ! $query->is_main_query() ) {
        return;
    }

    if ( 'rating' === $query->get( 'orderby' ) ) {
        $query->set( 'meta_key', '_rating' );
        $query->set( 'orderby', 'meta_value_num' );
    }
}
```

---

## Snippet 16 — Modify Main Query on Archive Pages

```php
add_action( 'pre_get_posts', 'modify_archive_query' );
function modify_archive_query( WP_Query $query ): void {
    // Only affect the main query on the frontend
    if ( is_admin() || ! $query->is_main_query() ) {
        return;
    }

    // Show 20 posts on category archives
    if ( $query->is_category() ) {
        $query->set( 'posts_per_page', 20 );
    }

    // Exclude a category from homepage
    if ( $query->is_home() ) {
        $query->set( 'cat', '-5' ); // Exclude category ID 5
    }

    // Show custom post types in search results
    if ( $query->is_search() ) {
        $query->set( 'post_type', [ 'post', 'page', 'book' ] );
    }
}
```

---

## Snippet 17 — Add Custom Admin Footer Text

```php
add_filter( 'admin_footer_text', 'custom_admin_footer' );
function custom_admin_footer(): string {
    return sprintf(
        esc_html__( 'Thank you for using %s. Built with ❤️', 'my-plugin' ),
        '<strong>' . esc_html( get_bloginfo( 'name' ) ) . '</strong>'
    );
}
```

---

## Snippet 18 — Create Your Own Action Hook

```php
// In your plugin — fire a custom action
function my_plugin_process_order( int $order_id ): void {
    // Do core processing...

    // Fire custom action so others can hook in
    do_action( 'my_plugin_order_processed', $order_id );
}

// Other plugins/themes can now hook in
add_action( 'my_plugin_order_processed', 'send_order_confirmation_email' );
function send_order_confirmation_email( int $order_id ): void {
    // Send email...
}
```

---

## Snippet 19 — Create Your Own Filter Hook

```php
// In your plugin — allow filtering of a value
function my_plugin_get_price( int $product_id ): float {
    $price = (float) get_post_meta( $product_id, '_price', true );

    // Allow others to modify the price
    return (float) apply_filters( 'my_plugin_product_price', $price, $product_id );
}

// Another plugin can now modify the price
add_filter( 'my_plugin_product_price', 'apply_member_discount', 10, 2 );
function apply_member_discount( float $price, int $product_id ): float {
    if ( is_user_logged_in() ) {
        return $price * 0.9; // 10% discount for logged-in users
    }
    return $price;
}
```

---

## Snippet 20 — Remove a Hook Added by a Plugin

```php
// Remove a hook added with a named function
remove_action( 'wp_head', 'wp_generator' );

// Remove a hook added with a class method (must match priority!)
add_action( 'plugins_loaded', function() {
    global $some_plugin;
    remove_action( 'init', [ $some_plugin, 'init_method' ], 10 );
});

// Remove a hook added with a singleton class
remove_filter( 'the_content', [ SomePlugin::get_instance(), 'modify_content' ], 20 );
```

---

## Snippet 21 — Add Async or Defer to Scripts

```php
add_filter( 'script_loader_tag', 'add_async_defer_attributes', 10, 3 );
function add_async_defer_attributes( string $tag, string $handle, string $src ): string {
    $async_scripts = [ 'google-analytics', 'facebook-pixel' ];
    $defer_scripts = [ 'my-non-critical-script', 'chat-widget' ];

    if ( in_array( $handle, $async_scripts, true ) ) {
        return str_replace( ' src=', ' async src=', $tag );
    }

    if ( in_array( $handle, $defer_scripts, true ) ) {
        return str_replace( ' src=', ' defer src=', $tag );
    }

    return $tag;
}
```

---

## Snippet 22 — Filter Email From Name & Address

```php
add_filter( 'wp_mail_from', fn() => 'noreply@yourdomain.com' );
add_filter( 'wp_mail_from_name', fn() => get_bloginfo( 'name' ) );
```

---

## Snippet 23 — Add Custom Cron Schedule

```php
// Register custom interval
add_filter( 'cron_schedules', 'add_custom_cron_schedule' );
function add_custom_cron_schedule( array $schedules ): array {
    $schedules['every_6_hours'] = [
        'interval' => 6 * HOUR_IN_SECONDS,
        'display'  => __( 'Every 6 Hours', 'my-plugin' ),
    ];
    return $schedules;
}

// Schedule the event on plugin activation
register_activation_hook( __FILE__, 'schedule_my_cron_job' );
function schedule_my_cron_job(): void {
    if ( ! wp_next_scheduled( 'my_cron_hook' ) ) {
        wp_schedule_event( time(), 'every_6_hours', 'my_cron_hook' );
    }
}

// Hook your function to the cron event
add_action( 'my_cron_hook', 'run_my_cron_task' );
function run_my_cron_task(): void {
    // Your scheduled task here...
}

// Always unschedule on deactivation
register_deactivation_hook( __FILE__, 'unschedule_my_cron_job' );
function unschedule_my_cron_job(): void {
    wp_clear_scheduled_hook( 'my_cron_hook' );
}
```

---

## Snippet 24 — Disable Comments Sitewide

```php
// Close comments on all posts
add_filter( 'comments_open', '__return_false', 20 );
add_filter( 'pings_open', '__return_false', 20 );

// Hide existing comments
add_filter( 'comments_array', '__return_empty_array', 10 );

// Remove comments from admin menu
add_action( 'admin_menu', function() {
    remove_menu_page( 'edit-comments.php' );
});

// Remove comments from admin bar
add_action( 'init', function() {
    if ( is_admin_bar_showing() ) {
        remove_action( 'admin_bar_menu', 'wp_admin_bar_comments_menu', 60 );
    }
});
```

---

## Snippet 25 — Priority & Accepted Args Cheat Sheet

```php
// Priority controls execution order — lower runs earlier
// Default priority is always 10

add_action( 'init', 'runs_very_early', 1 );
add_action( 'init', 'runs_early', 5 );
add_action( 'init', 'runs_at_default', 10 );  // Default
add_action( 'init', 'runs_late', 20 );
add_action( 'init', 'runs_very_late', 999 );  // Useful for overriding plugins

// Accepted args — how many parameters your callback receives
// Default is 1 — increase when you need extra arguments from the hook

add_filter( 'the_title', 'my_title_filter', 10, 2 ); // Receive $title + $post_id
function my_title_filter( string $title, int $post_id ): string {
    if ( 42 === $post_id ) {
        return 'Special Title';
    }
    return $title;
}

// Common __return_ helpers (avoid writing one-liner functions)
add_filter( 'show_admin_bar', '__return_false' );   // Return false
add_filter( 'comments_open', '__return_true' );     // Return true
add_filter( 'the_content', '__return_empty_string' ); // Return ''
add_filter( 'some_filter', '__return_zero' );       // Return 0
add_filter( 'some_filter', '__return_empty_array' ); // Return []
add_filter( 'some_filter', '__return_null' );       // Return null
```
