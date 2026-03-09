# 🔍 WP_Query & Database

> 25 battle-tested snippets for querying posts, users, terms, and the database directly.

---

## Snippet 1 — The Perfect WP_Query Template

```php
$args = [
    'post_type'      => 'post',
    'post_status'    => 'publish',
    'posts_per_page' => 10,
    'paged'          => get_query_var( 'paged', 1 ),
    'orderby'        => 'date',
    'order'          => 'DESC',
];

$query = new WP_Query( $args );

if ( $query->have_posts() ) :
    while ( $query->have_posts() ) : $query->the_post();
        // Your loop here
        the_title( '<h2>', '</h2>' );
    endwhile;
    wp_reset_postdata(); // ALWAYS reset after custom query
else :
    echo '<p>' . esc_html__( 'No posts found.', 'my-theme' ) . '</p>';
endif;
```

---

## Snippet 2 — Query Multiple Post Types

```php
$args = [
    'post_type'      => [ 'post', 'book', 'movie' ],
    'post_status'    => 'publish',
    'posts_per_page' => 12,
    'orderby'        => 'date',
    'order'          => 'DESC',
];

$query = new WP_Query( $args );
```

---

## Snippet 3 — Query by Meta Value

```php
// Single meta condition
$args = [
    'post_type'  => 'post',
    'meta_query' => [
        [
            'key'     => '_is_featured',
            'value'   => '1',
            'compare' => '=',
        ],
    ],
];

// Meta value between a range (e.g. price)
$args = [
    'post_type'  => 'product',
    'meta_query' => [
        [
            'key'     => '_price',
            'value'   => [ 10, 100 ],
            'type'    => 'NUMERIC',
            'compare' => 'BETWEEN',
        ],
    ],
];

// Meta key exists (any value)
$args = [
    'post_type'  => 'post',
    'meta_query' => [
        [
            'key'     => '_thumbnail_id',
            'compare' => 'EXISTS',
        ],
    ],
];
```

---

## Snippet 4 — Query by Taxonomy

```php
// Posts in a single category
$args = [
    'post_type' => 'post',
    'tax_query' => [
        [
            'taxonomy' => 'category',
            'field'    => 'slug',
            'terms'    => 'news',
        ],
    ],
];

// Posts in multiple tags (OR)
$args = [
    'post_type' => 'post',
    'tax_query' => [
        [
            'taxonomy' => 'post_tag',
            'field'    => 'slug',
            'terms'    => [ 'featured', 'popular' ],
            'operator' => 'IN', // IN = OR logic
        ],
    ],
];

// Posts NOT in a category
$args = [
    'post_type' => 'post',
    'tax_query' => [
        [
            'taxonomy' => 'category',
            'field'    => 'slug',
            'terms'    => 'uncategorized',
            'operator' => 'NOT IN',
        ],
    ],
];
```

---

## Snippet 5 — AND Logic Across Two Taxonomies

```php
$args = [
    'post_type' => 'post',
    'tax_query' => [
        'relation' => 'AND',
        [
            'taxonomy' => 'category',
            'field'    => 'slug',
            'terms'    => 'news',
        ],
        [
            'taxonomy' => 'post_tag',
            'field'    => 'slug',
            'terms'    => 'featured',
        ],
    ],
];
```

---

## Snippet 6 — Query by Date Range

```php
$args = [
    'post_type'  => 'post',
    'date_query' => [
        [
            'after'     => '2024-01-01',
            'before'    => '2024-12-31',
            'inclusive' => true,
        ],
    ],
];

// Posts from the last 30 days
$args = [
    'post_type'  => 'post',
    'date_query' => [
        [
            'after' => date( 'Y-m-d', strtotime( '-30 days' ) ),
        ],
    ],
];
```

---

## Snippet 7 — Query by Author

```php
// Single author by ID
$args = [
    'post_type' => 'post',
    'author'    => 5,
];

// Multiple authors
$args = [
    'post_type'  => 'post',
    'author__in' => [ 1, 3, 5 ],
];

// Exclude an author
$args = [
    'post_type'      => 'post',
    'author__not_in' => [ 2 ],
];
```

---

## Snippet 8 — Order by Meta Value

```php
// Order by numeric meta value (e.g. price, rating)
$args = [
    'post_type'  => 'product',
    'meta_key'   => '_price',
    'orderby'    => 'meta_value_num',
    'order'      => 'ASC',
];

// Order by string meta value
$args = [
    'post_type' => 'book',
    'meta_key'  => '_author_name',
    'orderby'   => 'meta_value',
    'order'     => 'ASC',
];
```

---

## Snippet 9 — Order by Multiple Fields

```php
$args = [
    'post_type' => 'post',
    'orderby'   => [
        'menu_order' => 'ASC',
        'date'       => 'DESC',
        'title'      => 'ASC',
    ],
];
```

---

## Snippet 10 — Get Specific Posts by IDs (Preserve Order)

```php
$ids = [ 5, 2, 8, 1, 9 ];

$args = [
    'post_type'              => 'any',
    'post__in'               => $ids,
    'orderby'                => 'post__in', // Preserve the array order
    'posts_per_page'         => count( $ids ),
    'no_found_rows'          => true,
    'update_post_meta_cache' => false,
];

$query = new WP_Query( $args );
```

---

## Snippet 11 — Exclude Posts by IDs

```php
$args = [
    'post_type'      => 'post',
    'post__not_in'   => [ 3, 7, 12 ],
    'posts_per_page' => 10,
];
```

---

## Snippet 12 — Optimised Query (No Pagination Count)

```php
// Use no_found_rows when you don't need pagination
// Skips the expensive SQL COUNT(*) query
$args = [
    'post_type'              => 'post',
    'posts_per_page'         => 5,
    'no_found_rows'          => true,  // Skip COUNT — no pagination needed
    'update_post_meta_cache' => false, // Skip if you don't use post meta in the loop
    'update_post_term_cache' => false, // Skip if you don't use terms in the loop
];
```

---

## Snippet 13 — Get Only Post IDs (Fastest Query)

```php
$args = [
    'post_type'      => 'post',
    'posts_per_page' => -1,
    'fields'         => 'ids', // Returns array of IDs only — very fast
    'no_found_rows'  => true,
];

$post_ids = get_posts( $args ); // [ 1, 5, 9, 12, ... ]
```

---

## Snippet 14 — Pagination with WP_Query

```php
$paged = get_query_var( 'paged' ) ? absint( get_query_var( 'paged' ) ) : 1;

$args = [
    'post_type'      => 'post',
    'posts_per_page' => 10,
    'paged'          => $paged,
];

$query = new WP_Query( $args );

// Output posts...
if ( $query->have_posts() ) :
    while ( $query->have_posts() ) : $query->the_post();
        the_title( '<h2>', '</h2>' );
    endwhile;
    wp_reset_postdata();

    // Pagination links
    echo paginate_links([
        'total'   => $query->max_num_pages,
        'current' => $paged,
    ]);
endif;
```

---

## Snippet 15 — Search Query with Post Type Filter

```php
// Extend search to include custom fields
add_action( 'pre_get_posts', 'extend_search_query' );
function extend_search_query( WP_Query $query ): void {
    if ( ! $query->is_search() || ! $query->is_main_query() || is_admin() ) {
        return;
    }

    // Include custom post types in search
    $query->set( 'post_type', [ 'post', 'page', 'book' ] );
}
```

---

## Snippet 16 — get_posts() for Simple Queries

```php
// get_posts() is a simpler wrapper — great for non-paginated queries
$posts = get_posts([
    'post_type'   => 'post',
    'numberposts' => 5,
    'category'    => 3,
    'orderby'     => 'date',
    'order'       => 'DESC',
]);

foreach ( $posts as $post ) {
    setup_postdata( $post );
    the_title( '<h3>', '</h3>' );
}
wp_reset_postdata();
```

---

## Snippet 17 — Query Users

```php
// Get all editors and administrators
$users = get_users([
    'role__in' => [ 'editor', 'administrator' ],
    'orderby'  => 'display_name',
    'order'    => 'ASC',
    'fields'   => [ 'ID', 'display_name', 'user_email' ],
]);

foreach ( $users as $user ) {
    echo esc_html( $user->display_name ) . '<br>';
}

// Count users by role
$count = count_users();
echo 'Admins: ' . $count['avail_roles']['administrator'];
```

---

## Snippet 18 — Query Terms (Categories, Tags, Custom Taxonomies)

```php
// Get all categories with posts
$terms = get_terms([
    'taxonomy'   => 'category',
    'hide_empty' => true,
    'orderby'    => 'count',
    'order'      => 'DESC',
    'number'     => 10, // Limit to 10
]);

if ( ! empty( $terms ) && ! is_wp_error( $terms ) ) {
    foreach ( $terms as $term ) {
        printf(
            '<a href="%s">%s (%d)</a>',
            esc_url( get_term_link( $term ) ),
            esc_html( $term->name ),
            $term->count
        );
    }
}
```

---

## Snippet 19 — Direct Database Query with $wpdb

```php
global $wpdb;

// SELECT multiple rows
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT ID, post_title FROM {$wpdb->posts}
         WHERE post_status = %s AND post_author = %d
         ORDER BY post_date DESC LIMIT %d",
        'publish',
        get_current_user_id(),
        10
    ),
    ARRAY_A
);

// SELECT single row
$row = $wpdb->get_row(
    $wpdb->prepare(
        "SELECT * FROM {$wpdb->users} WHERE ID = %d",
        $user_id
    )
);

// SELECT single value
$count = $wpdb->get_var(
    $wpdb->prepare(
        "SELECT COUNT(*) FROM {$wpdb->posts} WHERE post_author = %d AND post_status = %s",
        $user_id,
        'publish'
    )
);
```

---

## Snippet 20 — Insert, Update & Delete with $wpdb

```php
global $wpdb;

// INSERT
$wpdb->insert(
    $wpdb->prefix . 'my_table',
    [
        'user_id'    => get_current_user_id(),
        'value'      => 'hello world',
        'created_at' => current_time( 'mysql' ),
    ],
    [ '%d', '%s', '%s' ] // Format for each value
);
$new_id = $wpdb->insert_id;

// UPDATE
$wpdb->update(
    $wpdb->prefix . 'my_table',
    [ 'value' => 'updated value' ],  // Data to set
    [ 'ID'    => $new_id ],          // WHERE clause
    [ '%s' ],                        // Data formats
    [ '%d' ]                         // WHERE formats
);

// DELETE
$wpdb->delete(
    $wpdb->prefix . 'my_table',
    [ 'user_id' => get_current_user_id() ],
    [ '%d' ]
);
```

---

## Snippet 21 — Create Custom Database Table on Plugin Activation

```php
register_activation_hook( __FILE__, 'create_my_custom_table' );
function create_my_custom_table(): void {
    global $wpdb;

    $table_name      = $wpdb->prefix . 'my_logs';
    $charset_collate = $wpdb->get_charset_collate();

    $sql = "CREATE TABLE IF NOT EXISTS {$table_name} (
        id bigint(20) NOT NULL AUTO_INCREMENT,
        user_id bigint(20) NOT NULL,
        action varchar(100) NOT NULL,
        created_at datetime DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (id),
        KEY user_id (user_id)
    ) {$charset_collate};";

    require_once ABSPATH . 'wp-admin/includes/upgrade.php';
    dbDelta( $sql ); // dbDelta handles CREATE and ALTER safely
}
```

---

## Snippet 22 — Count Posts by Post Type

```php
// Count published posts for any post type
$counts = wp_count_posts( 'post' );
echo 'Published posts: ' . $counts->publish;
echo 'Draft posts: '     . $counts->draft;
echo 'Pending posts: '   . $counts->pending;

// Count for custom post type
$book_counts = wp_count_posts( 'book' );
echo 'Published books: ' . $book_counts->publish;
```

---

## Snippet 23 — Get Adjacent Posts (Previous / Next)

```php
// Get next post (newer)
$next_post = get_next_post( true, '', 'category' ); // true = stay in same category
if ( $next_post ) {
    echo '<a href="' . esc_url( get_permalink( $next_post->ID ) ) . '">'
        . esc_html( get_the_title( $next_post->ID ) ) . '</a>';
}

// Get previous post (older)
$prev_post = get_previous_post( true, '', 'category' );
if ( $prev_post ) {
    echo '<a href="' . esc_url( get_permalink( $prev_post->ID ) ) . '">'
        . esc_html( get_the_title( $prev_post->ID ) ) . '</a>';
}
```

---

## Snippet 24 — WP_Query Orderby Options (Cheat Sheet)

```php
// All valid 'orderby' values:

'date'           // Post date (default)
'modified'       // Last modified date
'title'          // Post title (alphabetical)
'name'           // Post slug (alphabetical)
'ID'             // Post ID
'rand'           // Random order — avoid on large sites!
'comment_count'  // Number of comments
'menu_order'     // Page/attachment order
'meta_value'     // Meta value (string)
'meta_value_num' // Meta value (numeric) — use this for numbers
'post__in'       // Preserve order of post__in array
'relevance'      // Search relevance (only with 's' param)
'none'           // No order — fastest, use with no_found_rows

// Multiple orderby (WordPress 4.0+)
'orderby' => [
    'menu_order' => 'ASC',
    'title'      => 'ASC',
],
```

---

## Snippet 25 — Cache WP_Query Results with Transients

```php
function get_featured_posts(): array {
    $cache_key = 'featured_posts_v1';
    $cached    = get_transient( $cache_key );

    if ( false !== $cached ) {
        return $cached;
    }

    $query = new WP_Query([
        'post_type'              => 'post',
        'posts_per_page'         => 6,
        'meta_key'               => '_is_featured',
        'meta_value'             => '1',
        'no_found_rows'          => true,
        'update_post_meta_cache' => false,
        'update_post_term_cache' => false,
    ]);

    $posts = $query->posts;
    set_transient( $cache_key, $posts, HOUR_IN_SECONDS );

    return $posts;
}

// Clear cache when any post is saved
add_action( 'save_post', fn() => delete_transient( 'featured_posts_v1' ) );
```
