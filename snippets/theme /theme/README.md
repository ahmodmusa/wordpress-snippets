# 🎨 Theme Development

> 25 snippets for building professional WordPress themes — setup, customizer, menus, template hierarchy, and more.

---

## Snippet 1 — functions.php Boilerplate

```php
<?php
if ( ! defined( 'ABSPATH' ) ) exit;

define( 'MY_THEME_VERSION', '1.0.0' );
define( 'MY_THEME_DIR', get_template_directory() );
define( 'MY_THEME_URI', get_template_directory_uri() );

add_action( 'after_setup_theme', 'my_theme_setup' );
function my_theme_setup(): void {
    load_theme_textdomain( 'my-theme', MY_THEME_DIR . '/languages' );

    add_theme_support( 'title-tag' );
    add_theme_support( 'post-thumbnails' );
    add_theme_support( 'automatic-feed-links' );
    add_theme_support( 'html5', [
        'search-form', 'comment-form', 'comment-list',
        'gallery', 'caption', 'style', 'script',
    ]);
    add_theme_support( 'customize-selective-refresh-widgets' );
    add_theme_support( 'wp-block-styles' );
    add_theme_support( 'align-wide' );
    add_theme_support( 'responsive-embeds' );
    add_theme_support( 'editor-styles' );

    register_nav_menus([
        'primary' => __( 'Primary Navigation', 'my-theme' ),
        'footer'  => __( 'Footer Navigation', 'my-theme' ),
        'mobile'  => __( 'Mobile Navigation', 'my-theme' ),
    ]);

    // Set default thumbnail size
    set_post_thumbnail_size( 1200, 628, true );
}
```

---

## Snippet 2 — Enqueue Scripts & Styles with Versioning

```php
add_action( 'wp_enqueue_scripts', 'my_theme_assets' );
function my_theme_assets(): void {
    // Main stylesheet — use filemtime for automatic cache busting
    wp_enqueue_style(
        'my-theme-style',
        MY_THEME_URI . '/css/style.css',
        [],
        filemtime( MY_THEME_DIR . '/css/style.css' )
    );

    // Main JS — load in footer
    wp_enqueue_script(
        'my-theme-script',
        MY_THEME_URI . '/js/main.js',
        [],
        filemtime( MY_THEME_DIR . '/js/main.js' ),
        true
    );

    // Pass data to JS
    wp_localize_script( 'my-theme-script', 'ThemeData', [
        'ajax_url' => admin_url( 'admin-ajax.php' ),
        'nonce'    => wp_create_nonce( 'theme_nonce' ),
        'home_url' => home_url(),
        'is_rtl'   => is_rtl() ? 'true' : 'false',
    ]);

    // Enqueue comments reply script on single posts with comments open
    if ( is_singular() && comments_open() && get_option( 'thread_comments' ) ) {
        wp_enqueue_script( 'comment-reply' );
    }
}
```

---

## Snippet 3 — Register Custom Image Sizes

```php
add_action( 'after_setup_theme', 'register_my_image_sizes' );
function register_my_image_sizes(): void {
    // Hard crop (exact dimensions)
    add_image_size( 'hero',      1920, 600,  true );
    add_image_size( 'card',      600,  400,  true );
    add_image_size( 'square',    400,  400,  true );
    add_image_size( 'og-image',  1200, 630,  true );

    // Soft crop (proportional, no crop)
    add_image_size( 'wide',      1200, 9999, false );
    add_image_size( 'portrait',  600,  900,  false );
}

// Make custom sizes selectable in the media library
add_filter( 'image_size_names_choose', 'add_custom_image_sizes_to_admin' );
function add_custom_image_sizes_to_admin( array $sizes ): array {
    return array_merge( $sizes, [
        'hero'     => __( 'Hero',     'my-theme' ),
        'card'     => __( 'Card',     'my-theme' ),
        'square'   => __( 'Square',   'my-theme' ),
        'og-image' => __( 'OG Image', 'my-theme' ),
    ]);
}
```

---

## Snippet 4 — Register Widget Areas (Sidebars)

```php
add_action( 'widgets_init', 'register_my_sidebars' );
function register_my_sidebars(): void {
    $defaults = [
        'before_widget' => '<section id="%1$s" class="widget %2$s">',
        'after_widget'  => '</section>',
        'before_title'  => '<h3 class="widget-title">',
        'after_title'   => '</h3>',
    ];

    register_sidebar( array_merge( $defaults, [
        'name'        => __( 'Main Sidebar', 'my-theme' ),
        'id'          => 'sidebar-main',
        'description' => __( 'Appears on posts and pages.', 'my-theme' ),
    ]));

    register_sidebar( array_merge( $defaults, [
        'name'        => __( 'Footer — Column 1', 'my-theme' ),
        'id'          => 'footer-col-1',
        'description' => __( 'First footer column.', 'my-theme' ),
    ]));

    register_sidebar( array_merge( $defaults, [
        'name'        => __( 'Footer — Column 2', 'my-theme' ),
        'id'          => 'footer-col-2',
        'description' => __( 'Second footer column.', 'my-theme' ),
    ]));
}
```

---

## Snippet 5 — Display a Navigation Menu

```php
// Display primary nav with custom walker
wp_nav_menu([
    'theme_location' => 'primary',
    'menu_class'     => 'nav-menu',
    'container'      => 'nav',
    'container_class'=> 'site-navigation',
    'fallback_cb'    => false,
    'depth'          => 2,
    'items_wrap'     => '<ul id="%1$s" class="%2$s">%3$s</ul>',
]);

// Check if menu is assigned before displaying
if ( has_nav_menu( 'primary' ) ) {
    wp_nav_menu([ 'theme_location' => 'primary' ]);
}
```

---

## Snippet 6 — Add Custom Fields to Nav Menu Items

```php
// Add a custom field to menu items in the admin
add_filter( 'wp_nav_menu_item_custom_fields', 'add_menu_item_badge_field', 10, 2 );
function add_menu_item_badge_field( string $item_id, WP_Post $item ): void {
    $badge = get_post_meta( $item->ID, '_menu_badge', true );
    ?>
    <p class="field-badge description description-wide">
        <label for="edit-menu-item-badge-<?php echo esc_attr( $item_id ); ?>">
            <?php esc_html_e( 'Badge Text', 'my-theme' ); ?>
            <input type="text"
                   name="menu-item-badge[<?php echo esc_attr( $item_id ); ?>]"
                   value="<?php echo esc_attr( $badge ); ?>"
                   class="widefat">
        </label>
    </p>
    <?php
}

// Save the field
add_action( 'wp_update_nav_menu_item', 'save_menu_item_badge_field', 10, 2 );
function save_menu_item_badge_field( int $menu_id, int $menu_item_db_id ): void {
    if ( isset( $_POST['menu-item-badge'][ $menu_item_db_id ] ) ) {
        update_post_meta(
            $menu_item_db_id,
            '_menu_badge',
            sanitize_text_field( wp_unslash( $_POST['menu-item-badge'][ $menu_item_db_id ] ) )
        );
    }
}
```

---

## Snippet 7 — Customizer: Add Settings, Sections & Controls

```php
add_action( 'customize_register', 'my_theme_customizer' );
function my_theme_customizer( WP_Customize_Manager $wp_customize ): void {
    // Add a new section
    $wp_customize->add_section( 'my_theme_options', [
        'title'    => __( 'Theme Options', 'my-theme' ),
        'priority' => 30,
    ]);

    // Text setting
    $wp_customize->add_setting( 'phone_number', [
        'default'           => '',
        'sanitize_callback' => 'sanitize_text_field',
        'transport'         => 'postMessage', // Live preview without reload
    ]);
    $wp_customize->add_control( 'phone_number', [
        'label'   => __( 'Phone Number', 'my-theme' ),
        'section' => 'my_theme_options',
        'type'    => 'text',
    ]);

    // Color setting
    $wp_customize->add_setting( 'primary_color', [
        'default'           => '#0073aa',
        'sanitize_callback' => 'sanitize_hex_color',
        'transport'         => 'postMessage',
    ]);
    $wp_customize->add_control(
        new WP_Customize_Color_Control( $wp_customize, 'primary_color', [
            'label'   => __( 'Primary Colour', 'my-theme' ),
            'section' => 'my_theme_options',
        ])
    );

    // Select setting
    $wp_customize->add_setting( 'layout_style', [
        'default'           => 'boxed',
        'sanitize_callback' => fn( $v ) => in_array( $v, [ 'boxed', 'full-width' ], true ) ? $v : 'boxed',
    ]);
    $wp_customize->add_control( 'layout_style', [
        'label'   => __( 'Layout Style', 'my-theme' ),
        'section' => 'my_theme_options',
        'type'    => 'select',
        'choices' => [
            'boxed'      => __( 'Boxed', 'my-theme' ),
            'full-width' => __( 'Full Width', 'my-theme' ),
        ],
    ]);
}
```

---

## Snippet 8 — Output Customizer CSS Dynamically

```php
add_action( 'wp_head', 'output_customizer_css' );
function output_customizer_css(): void {
    $primary_color = sanitize_hex_color( get_theme_mod( 'primary_color', '#0073aa' ) );
    ?>
    <style id="my-theme-custom-css">
        :root {
            --color-primary: <?php echo esc_attr( $primary_color ); ?>;
        }
        a, a:hover { color: var(--color-primary); }
        .btn-primary { background-color: var(--color-primary); }
    </style>
    <?php
}
```

---

## Snippet 9 — Template Hierarchy Cheat Sheet

```php
/**
 * WordPress Template Hierarchy (simplified)
 *
 * Single post:
 *   single-{post-type}-{slug}.php → single-{post-type}.php → single.php → singular.php → index.php
 *
 * Page:
 *   {custom-template}.php → page-{slug}.php → page-{id}.php → page.php → singular.php → index.php
 *
 * Category archive:
 *   category-{slug}.php → category-{id}.php → category.php → archive.php → index.php
 *
 * Custom post type archive:
 *   archive-{post-type}.php → archive.php → index.php
 *
 * Author archive:
 *   author-{nicename}.php → author-{id}.php → author.php → archive.php → index.php
 *
 * Search results:
 *   search.php → index.php
 *
 * 404:
 *   404.php → index.php
 *
 * Front page:
 *   front-page.php → home.php (if static) / home.php (if posts) → index.php
 */
```

---

## Snippet 10 — Breadcrumbs Without a Plugin

```php
function my_breadcrumbs(): void {
    if ( is_front_page() ) return;

    echo '<nav class="breadcrumbs" aria-label="Breadcrumb"><ol>';

    echo '<li><a href="' . esc_url( home_url() ) . '">' . esc_html__( 'Home', 'my-theme' ) . '</a></li>';

    if ( is_category() || is_single() ) {
        $categories = get_the_category();
        if ( $categories ) {
            echo '<li><a href="' . esc_url( get_category_link( $categories[0]->term_id ) ) . '">'
                . esc_html( $categories[0]->name ) . '</a></li>';
        }
        if ( is_single() ) {
            echo '<li aria-current="page">' . esc_html( get_the_title() ) . '</li>';
        }
    } elseif ( is_page() ) {
        if ( wp_get_post_parent_id( get_the_ID() ) ) {
            echo '<li><a href="' . esc_url( get_permalink( wp_get_post_parent_id( get_the_ID() ) ) ) . '">'
                . esc_html( get_the_title( wp_get_post_parent_id( get_the_ID() ) ) ) . '</a></li>';
        }
        echo '<li aria-current="page">' . esc_html( get_the_title() ) . '</li>';
    } elseif ( is_search() ) {
        echo '<li aria-current="page">'
            . sprintf( esc_html__( 'Search: %s', 'my-theme' ), esc_html( get_search_query() ) )
            . '</li>';
    } elseif ( is_404() ) {
        echo '<li aria-current="page">' . esc_html__( '404 Not Found', 'my-theme' ) . '</li>';
    } elseif ( is_archive() ) {
        echo '<li aria-current="page">' . esc_html( get_the_archive_title() ) . '</li>';
    }

    echo '</ol></nav>';
}
```

---

## Snippet 11 — Pagination Without a Plugin

```php
function my_pagination( WP_Query $query = null ): void {
    global $wp_query;
    $query = $query ?? $wp_query;

    if ( $query->max_num_pages <= 1 ) return;

    $paged = get_query_var( 'paged' ) ?: 1;

    echo '<nav class="pagination" aria-label="Posts navigation">';
    echo paginate_links([
        'base'      => str_replace( PHP_INT_MAX, '%#%', esc_url( get_pagenum_link( PHP_INT_MAX ) ) ),
        'format'    => '?paged=%#%',
        'current'   => $paged,
        'total'     => $query->max_num_pages,
        'prev_text' => '&larr; ' . __( 'Previous', 'my-theme' ),
        'next_text' => __( 'Next', 'my-theme' ) . ' &rarr;',
        'type'      => 'list',
        'end_size'  => 2,
        'mid_size'  => 1,
    ]);
    echo '</nav>';
}
```

---

## Snippet 12 — Custom Post Type with All Options

```php
add_action( 'init', 'register_book_post_type' );
function register_book_post_type(): void {
    $labels = [
        'name'               => __( 'Books', 'my-theme' ),
        'singular_name'      => __( 'Book', 'my-theme' ),
        'add_new_item'       => __( 'Add New Book', 'my-theme' ),
        'edit_item'          => __( 'Edit Book', 'my-theme' ),
        'new_item'           => __( 'New Book', 'my-theme' ),
        'view_item'          => __( 'View Book', 'my-theme' ),
        'search_items'       => __( 'Search Books', 'my-theme' ),
        'not_found'          => __( 'No books found', 'my-theme' ),
        'not_found_in_trash' => __( 'No books in trash', 'my-theme' ),
        'menu_name'          => __( 'Books', 'my-theme' ),
    ];

    register_post_type( 'book', [
        'labels'        => $labels,
        'public'        => true,
        'has_archive'   => true,
        'rewrite'       => [ 'slug' => 'books', 'with_front' => false ],
        'supports'      => [ 'title', 'editor', 'thumbnail', 'excerpt', 'custom-fields', 'author' ],
        'show_in_rest'  => true,
        'menu_icon'     => 'dashicons-book-alt',
        'menu_position' => 5,
        'taxonomies'    => [ 'genre' ],
    ]);
}
```

---

## Snippet 13 — Custom Taxonomy

```php
add_action( 'init', 'register_genre_taxonomy' );
function register_genre_taxonomy(): void {
    $labels = [
        'name'              => __( 'Genres', 'my-theme' ),
        'singular_name'     => __( 'Genre', 'my-theme' ),
        'search_items'      => __( 'Search Genres', 'my-theme' ),
        'all_items'         => __( 'All Genres', 'my-theme' ),
        'edit_item'         => __( 'Edit Genre', 'my-theme' ),
        'update_item'       => __( 'Update Genre', 'my-theme' ),
        'add_new_item'      => __( 'Add New Genre', 'my-theme' ),
        'new_item_name'     => __( 'New Genre Name', 'my-theme' ),
        'menu_name'         => __( 'Genre', 'my-theme' ),
    ];

    register_taxonomy( 'genre', [ 'book' ], [
        'labels'            => $labels,
        'hierarchical'      => true,  // true = like categories, false = like tags
        'public'            => true,
        'show_in_rest'      => true,
        'show_admin_column' => true,
        'rewrite'           => [ 'slug' => 'genre' ],
    ]);
}
```

---

## Snippet 14 — Custom Meta Box

```php
// Add meta box
add_action( 'add_meta_boxes', 'add_book_meta_box' );
function add_book_meta_box(): void {
    add_meta_box(
        'book_details',
        __( 'Book Details', 'my-theme' ),
        'render_book_meta_box',
        'book',
        'normal',
        'high'
    );
}

// Render meta box HTML
function render_book_meta_box( WP_Post $post ): void {
    wp_nonce_field( 'save_book_meta', 'book_meta_nonce' );
    $isbn = get_post_meta( $post->ID, '_isbn', true );
    ?>
    <p>
        <label for="book_isbn"><?php esc_html_e( 'ISBN', 'my-theme' ); ?></label>
        <input type="text" id="book_isbn" name="book_isbn"
               value="<?php echo esc_attr( $isbn ); ?>" class="widefat">
    </p>
    <?php
}

// Save meta box data
add_action( 'save_post_book', 'save_book_meta_box' );
function save_book_meta_box( int $post_id ): void {
    if ( ! isset( $_POST['book_meta_nonce'] ) ||
         ! wp_verify_nonce( sanitize_key( $_POST['book_meta_nonce'] ), 'save_book_meta' ) ) return;
    if ( defined( 'DOING_AUTOSAVE' ) && DOING_AUTOSAVE ) return;
    if ( ! current_user_can( 'edit_post', $post_id ) ) return;

    if ( isset( $_POST['book_isbn'] ) ) {
        update_post_meta( $post_id, '_isbn', sanitize_text_field( wp_unslash( $_POST['book_isbn'] ) ) );
    }
}
```

---

## Snippet 15 — Add Open Graph Meta Tags

```php
add_action( 'wp_head', 'output_open_graph_tags' );
function output_open_graph_tags(): void {
    if ( ! is_singular() ) return;

    $title       = get_the_title();
    $description = wp_trim_words( get_the_excerpt(), 30 );
    $url         = get_permalink();
    $image       = get_the_post_thumbnail_url( null, 'og-image' ) ?: get_site_icon_url( 1200 );
    $type        = 'article';

    echo '<meta property="og:title" content="'       . esc_attr( $title )       . '">' . "\n";
    echo '<meta property="og:description" content="' . esc_attr( $description ) . '">' . "\n";
    echo '<meta property="og:url" content="'         . esc_url( $url )          . '">' . "\n";
    echo '<meta property="og:type" content="'        . esc_attr( $type )        . '">' . "\n";
    echo '<meta property="og:site_name" content="'   . esc_attr( get_bloginfo( 'name' ) ) . '">' . "\n";

    if ( $image ) {
        echo '<meta property="og:image" content="' . esc_url( $image ) . '">' . "\n";
        echo '<meta name="twitter:card" content="summary_large_image">' . "\n";
    }

    echo '<meta name="twitter:title" content="'       . esc_attr( $title )       . '">' . "\n";
    echo '<meta name="twitter:description" content="' . esc_attr( $description ) . '">' . "\n";
}
```

---

## Snippet 16 — Theme Editor Style (Editor Matches Frontend)

```php
add_action( 'after_setup_theme', function() {
    add_theme_support( 'editor-styles' );
    add_editor_style( 'css/editor-style.css' );
});
```

---

## Snippet 17 — Defer Non-Critical Styles (Load CSS Async)

```php
add_filter( 'style_loader_tag', 'defer_non_critical_styles', 10, 4 );
function defer_non_critical_styles( string $html, string $handle, string $href, string $media ): string {
    $defer_styles = [ 'fontawesome', 'my-slick-slider', 'my-lightbox' ];

    if ( is_admin() || ! in_array( $handle, $defer_styles, true ) ) {
        return $html;
    }

    // Load asynchronously
    return str_replace(
        "rel='stylesheet'",
        "rel='preload' as='style' onload=\"this.onload=null;this.rel='stylesheet'\"",
        $html
    ) . "<noscript>{$html}</noscript>";
}
```

---

## Snippet 18 — Template Part with Context (WordPress 5.5+)

```php
// Pass data to template parts using the $args parameter
get_template_part( 'template-parts/card', 'post', [
    'post_id'     => get_the_ID(),
    'show_author' => true,
    'image_size'  => 'card',
]);

// Inside template-parts/card-post.php:
// $args['post_id'], $args['show_author'], $args['image_size'] are available
$post_id     = absint( $args['post_id'] ?? get_the_ID() );
$show_author = (bool) ( $args['show_author'] ?? false );
$image_size  = sanitize_key( $args['image_size'] ?? 'medium' );
```

---

## Snippet 19 — Add Schema.org Structured Data

```php
add_action( 'wp_head', 'output_schema_markup' );
function output_schema_markup(): void {
    if ( ! is_singular( 'post' ) ) return;

    $schema = [
        '@context'  => 'https://schema.org',
        '@type'     => 'Article',
        'headline'  => get_the_title(),
        'datePublished' => get_the_date( 'c' ),
        'dateModified'  => get_the_modified_date( 'c' ),
        'author'    => [
            '@type' => 'Person',
            'name'  => get_the_author(),
        ],
        'publisher' => [
            '@type' => 'Organization',
            'name'  => get_bloginfo( 'name' ),
            'logo'  => [
                '@type' => 'ImageObject',
                'url'   => get_site_icon_url( 512 ),
            ],
        ],
    ];

    if ( has_post_thumbnail() ) {
        $schema['image'] = get_the_post_thumbnail_url( null, 'large' );
    }

    echo '<script type="application/ld+json">'
        . wp_json_encode( $schema, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE )
        . '</script>' . "\n";
}
```

---

## Snippet 20 — Reading Time Function

```php
function get_reading_time( int $post_id = 0 ): string {
    $post_id    = $post_id ?: get_the_ID();
    $content    = get_post_field( 'post_content', $post_id );
    $word_count = str_word_count( wp_strip_all_tags( $content ) );
    $minutes    = max( 1, (int) ceil( $word_count / 200 ) );

    return sprintf(
        /* translators: %d: number of minutes */
        _n( '%d min read', '%d min read', $minutes, 'my-theme' ),
        $minutes
    );
}

// Usage in template:
echo esc_html( get_reading_time() );
```

---

## Snippet 21 — Get Related Posts by Category

```php
function get_related_posts( int $post_id = 0, int $count = 3 ): array {
    $post_id    = $post_id ?: get_the_ID();
    $categories = wp_get_post_categories( $post_id );

    if ( empty( $categories ) ) {
        return [];
    }

    $related = get_posts([
        'post_type'      => get_post_type( $post_id ),
        'posts_per_page' => $count,
        'post__not_in'   => [ $post_id ],
        'no_found_rows'  => true,
        'tax_query'      => [[
            'taxonomy' => 'category',
            'field'    => 'term_id',
            'terms'    => $categories,
        ]],
    ]);

    return $related;
}
```

---

## Snippet 22 — Custom Login Page Styles

```php
add_action( 'login_enqueue_scripts', 'custom_login_styles' );
function custom_login_styles(): void {
    wp_enqueue_style(
        'custom-login',
        MY_THEME_URI . '/css/login.css',
        [],
        filemtime( MY_THEME_DIR . '/css/login.css' )
    );
}

// Remove the "Back to {site}" link
add_filter( 'login_display_language_dropdown', '__return_false' );

// Change login page background
add_action( 'login_head', function() {
    ?>
    <style>
        body.login { background: #1a1a2e; }
        .login form { border-radius: 8px; box-shadow: 0 4px 24px rgba(0,0,0,.2); }
    </style>
    <?php
});
```

---

## Snippet 23 — Handle 404 Gracefully

```php
// Custom 404 search suggestion in 404.php template
function get_404_suggestions(): array {
    $search_term = sanitize_text_field( $_SERVER['REQUEST_URI'] ?? '' );
    $search_term = trim( str_replace( [ '/', '-', '_' ], ' ', $search_term ) );

    if ( empty( $search_term ) ) return [];

    return get_posts([
        's'              => $search_term,
        'numberposts'    => 5,
        'post_status'    => 'publish',
        'no_found_rows'  => true,
    ]);
}
```

---

## Snippet 24 — Disable Admin Bar on Frontend for Non-Admins

```php
add_action( 'after_setup_theme', 'hide_admin_bar_for_non_admins' );
function hide_admin_bar_for_non_admins(): void {
    if ( ! current_user_can( 'manage_options' ) ) {
        show_admin_bar( false );
    }
}
```

---

## Snippet 25 — Theme Development Checklist

```php
/**
 * WordPress Theme Development Checklist
 *
 * SETUP
 * ✅ add_theme_support() for: title-tag, post-thumbnails, html5, editor-styles
 * ✅ load_theme_textdomain() for translations
 * ✅ register_nav_menus() for all menu locations
 * ✅ register_sidebar() for all widget areas
 * ✅ Custom image sizes registered and added to media library chooser
 *
 * TEMPLATES
 * ✅ index.php exists (required)
 * ✅ style.css has correct theme header comment
 * ✅ wp_head() called in header.php before </head>
 * ✅ wp_footer() called in footer.php before </body>
 * ✅ body_class() used on <body> tag
 * ✅ post_class() used on article/post wrapper
 * ✅ wp_nav_menu() used for all menus (not hardcoded HTML)
 * ✅ Pagination implemented with paginate_links()
 *
 * SECURITY & STANDARDS
 * ✅ All output escaped: esc_html(), esc_url(), esc_attr()
 * ✅ All user input sanitized
 * ✅ Translation functions used: __(), _e(), esc_html__()
 * ✅ Nonces used on all forms
 * ✅ No hardcoded URLs — use home_url(), get_template_directory_uri()
 *
 * PERFORMANCE
 * ✅ Scripts enqueued in footer (last arg = true)
 * ✅ File versioning with filemtime() for cache busting
 * ✅ Emoji scripts removed if not needed
 *
 * SEO & ACCESSIBILITY
 * ✅ Open Graph tags in <head>
 * ✅ Schema.org markup on post types
 * ✅ Alt text on all images (alt="" for decorative)
 * ✅ Proper heading hierarchy (only one H1 per page)
 * ✅ Skip navigation link before header
 * ✅ ARIA labels on navigation elements
 */
```
