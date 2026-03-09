# 🔍 WP_Query & Database Snippets

---

## WP_Query Argument Cheat Sheet

```php
$args = [
    // Post type & status
    'post_type'   => 'post',               // string or array: ['post', 'page']
    'post_status' => 'publish',            // string or array

    // Pagination
    'posts_per_page' => 10,               // -1 for all
    'paged'          => get_query_var('paged', 1),
    'offset'         => 0,                // Avoid with pagination

    // Ordering
    'orderby' => 'date',                  // See orderby options below
    'order'   => 'DESC',                  // ASC or DESC

    // Author
    'author'    => 1,                     // User ID
    'author__in' => [1, 2, 3],

    // Post inclusion/exclusion
    'post__in'     => [1, 2, 3],
    'post__not_in' => [4, 5],

    // Meta query
    'meta_key'   => '_price',             // Simple meta filter
    'meta_value' => '100',
    'meta_query' => [ /* see below */ ],

    // Tax query
    'tax_query' => [ /* see below */ ],

    // Performance
    'no_found_rows'          => true,     // Disable pagination count
    'update_post_meta_cache' => false,    // Skip meta cache
    'update_post_term_cache' => false,    // Skip term cache
    'fields'                 => 'ids',   // Return IDs only
];
```

---

## Meta Query Examples

```php
// Single meta condition
'meta_query' => [
    [ 'key' => '_featured', 'value' => '1', 'compare' => '=' ],
],

// Multiple conditions (AND)
'meta_query' => [
    'relation' => 'AND',
    [ 'key' => '_price', 'value' => [ 10, 100 ], 'type' => 'NUMERIC', 'compare' => 'BETWEEN' ],
    [ 'key' => '_in_stock', 'value' => '1', 'compare' => '=' ],
],

// Multiple conditions (OR)
'meta_query' => [
    'relation' => 'OR',
    [ 'key' => '_sale', 'value' => '1' ],
    [ 'key' => '_new', 'value' => '1' ],
],

// Key exists (any value)
'meta_query' => [
    [ 'key' => '_thumbnail_id', 'compare' => 'EXISTS' ],
],
```

---

## Tax Query Examples

```php
// Posts in a category
'tax_query' => [
    [ 'taxonomy' => 'category', 'field' => 'slug', 'terms' => 'news' ],
],

// Posts in multiple categories (OR)
'tax_query' => [
    [ 'taxonomy' => 'category', 'field' => 'slug', 'terms' => ['news', 'updates'], 'operator' => 'IN' ],
],

// Posts NOT in a category
'tax_query' => [
    [ 'taxonomy' => 'category', 'field' => 'slug', 'terms' => 'uncategorized', 'operator' => 'NOT IN' ],
],

// Posts that have a term AND another term (different taxonomies)
'tax_query' => [
    'relation' => 'AND',
    [ 'taxonomy' => 'category', 'field' => 'slug', 'terms' => 'news' ],
    [ 'taxonomy' => 'post_tag', 'field' => 'slug', 'terms' => 'featured' ],
],
```

---

## Get Posts by Date Range

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
```

---

## Custom Database Queries

```php
global $wpdb;

// Get all with results
$rows = $wpdb->get_results(
    $wpdb->prepare( "SELECT * FROM {$wpdb->postmeta} WHERE meta_key = %s", '_my_key' ),
    ARRAY_A // OBJECT (default), ARRAY_A (assoc), ARRAY_N (numeric)
);

// Get single row
$row = $wpdb->get_row(
    $wpdb->prepare( "SELECT * FROM {$wpdb->users} WHERE ID = %d", $user_id )
);

// Get single value
$count = $wpdb->get_var(
    $wpdb->prepare( "SELECT COUNT(*) FROM {$wpdb->posts} WHERE post_author = %d", $user_id )
);

// Insert
$wpdb->insert(
    $wpdb->prefix . 'my_table',
    [ 'user_id' => 1, 'value' => 'hello' ],
    [ '%d', '%s' ]
);
$new_id = $wpdb->insert_id;

// Update
$wpdb->update(
    $wpdb->prefix . 'my_table',
    [ 'value' => 'updated' ],    // Data
    [ 'user_id' => 1 ],          // Where
    [ '%s' ],                    // Data formats
    [ '%d' ]                     // Where formats
);

// Delete
$wpdb->delete(
    $wpdb->prefix . 'my_table',
    [ 'user_id' => 1 ],
    [ '%d' ]
);
```
