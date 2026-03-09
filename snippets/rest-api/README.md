# 🌐 REST API

> 25 snippets for working with the WordPress REST API — custom endpoints, authentication, extending core routes, and more.

---

## How the WordPress REST API Works

```
Client → GET/POST https://yoursite.com/wp-json/namespace/v1/route
       → WordPress routes to registered callback
       → Callback returns WP_REST_Response or WP_Error
       → JSON response sent back
```

Base URL: `https://yoursite.com/wp-json/`
Namespace convention: `my-plugin/v1`

---

## Snippet 1 — Register a Basic GET Endpoint

```php
add_action( 'rest_api_init', 'register_my_rest_routes' );
function register_my_rest_routes(): void {
    register_rest_route( 'my-plugin/v1', '/hello', [
        'methods'             => WP_REST_Server::READABLE, // GET
        'callback'            => 'my_hello_endpoint',
        'permission_callback' => '__return_true', // Public endpoint
    ]);
}

function my_hello_endpoint( WP_REST_Request $request ): WP_REST_Response {
    return rest_ensure_response([
        'message' => 'Hello from the REST API!',
        'time'    => current_time( 'mysql' ),
    ]);
}
```

---

## Snippet 2 — GET Endpoint with URL Parameter

```php
add_action( 'rest_api_init', function() {
    register_rest_route( 'my-plugin/v1', '/posts/(?P<id>\d+)', [
        'methods'             => WP_REST_Server::READABLE,
        'callback'            => 'get_my_post',
        'permission_callback' => '__return_true',
        'args'                => [
            'id' => [
                'required'          => true,
                'validate_callback' => fn( $v ) => is_numeric( $v ) && $v > 0,
                'sanitize_callback' => 'absint',
                'description'       => 'Post ID',
            ],
        ],
    ]);
});

function get_my_post( WP_REST_Request $request ): WP_REST_Response|WP_Error {
    $post = get_post( $request->get_param( 'id' ) );

    if ( ! $post || 'publish' !== $post->post_status ) {
        return new WP_Error( 'not_found', 'Post not found.', [ 'status' => 404 ] );
    }

    return rest_ensure_response([
        'id'      => $post->ID,
        'title'   => get_the_title( $post ),
        'excerpt' => get_the_excerpt( $post ),
        'link'    => get_permalink( $post ),
        'date'    => $post->post_date,
    ]);
}
```

---

## Snippet 3 — POST Endpoint (Create Resource)

```php
add_action( 'rest_api_init', function() {
    register_rest_route( 'my-plugin/v1', '/notes', [
        'methods'             => WP_REST_Server::CREATABLE, // POST
        'callback'            => 'create_note',
        'permission_callback' => fn() => is_user_logged_in(),
        'args'                => [
            'title' => [
                'required'          => true,
                'type'              => 'string',
                'sanitize_callback' => 'sanitize_text_field',
            ],
            'content' => [
                'required'          => true,
                'type'              => 'string',
                'sanitize_callback' => 'wp_kses_post',
            ],
        ],
    ]);
});

function create_note( WP_REST_Request $request ): WP_REST_Response|WP_Error {
    $post_id = wp_insert_post([
        'post_title'   => $request->get_param( 'title' ),
        'post_content' => $request->get_param( 'content' ),
        'post_status'  => 'publish',
        'post_type'    => 'note',
        'post_author'  => get_current_user_id(),
    ], true );

    if ( is_wp_error( $post_id ) ) {
        return new WP_Error( 'insert_failed', $post_id->get_error_message(), [ 'status' => 500 ] );
    }

    return rest_ensure_response([
        'id'      => $post_id,
        'link'    => get_permalink( $post_id ),
        'message' => 'Note created.',
    ]);
}
```

---

## Snippet 4 — PUT / PATCH Endpoint (Update Resource)

```php
add_action( 'rest_api_init', function() {
    register_rest_route( 'my-plugin/v1', '/notes/(?P<id>\d+)', [
        'methods'             => WP_REST_Server::EDITABLE, // PUT + PATCH
        'callback'            => 'update_note',
        'permission_callback' => fn( $req ) => current_user_can( 'edit_post', (int) $req['id'] ),
        'args'                => [
            'id' => [
                'required'          => true,
                'sanitize_callback' => 'absint',
            ],
            'title' => [
                'type'              => 'string',
                'sanitize_callback' => 'sanitize_text_field',
            ],
        ],
    ]);
});

function update_note( WP_REST_Request $request ): WP_REST_Response|WP_Error {
    $post_id = $request->get_param( 'id' );
    $post    = get_post( $post_id );

    if ( ! $post ) {
        return new WP_Error( 'not_found', 'Note not found.', [ 'status' => 404 ] );
    }

    $update = [ 'ID' => $post_id ];

    if ( $request->has_param( 'title' ) ) {
        $update['post_title'] = $request->get_param( 'title' );
    }

    wp_update_post( $update );

    return rest_ensure_response( [ 'id' => $post_id, 'updated' => true ] );
}
```

---

## Snippet 5 — DELETE Endpoint

```php
add_action( 'rest_api_init', function() {
    register_rest_route( 'my-plugin/v1', '/notes/(?P<id>\d+)', [
        'methods'             => WP_REST_Server::DELETABLE, // DELETE
        'callback'            => 'delete_note',
        'permission_callback' => fn( $req ) => current_user_can( 'delete_post', (int) $req['id'] ),
        'args'                => [
            'id' => [ 'required' => true, 'sanitize_callback' => 'absint' ],
        ],
    ]);
});

function delete_note( WP_REST_Request $request ): WP_REST_Response|WP_Error {
    $post_id = $request->get_param( 'id' );

    if ( ! get_post( $post_id ) ) {
        return new WP_Error( 'not_found', 'Note not found.', [ 'status' => 404 ] );
    }

    $deleted = wp_delete_post( $post_id, true );

    if ( ! $deleted ) {
        return new WP_Error( 'delete_failed', 'Could not delete note.', [ 'status' => 500 ] );
    }

    return rest_ensure_response( [ 'deleted' => true, 'id' => $post_id ] );
}
```

---

## Snippet 6 — Register Multiple Methods on One Route

```php
add_action( 'rest_api_init', function() {
    $base = 'my-plugin/v1';
    $path = '/items/(?P<id>\d+)';

    // GET
    register_rest_route( $base, $path, [
        'methods'             => WP_REST_Server::READABLE,
        'callback'            => 'rest_get_item',
        'permission_callback' => '__return_true',
        'args'                => [ 'id' => [ 'sanitize_callback' => 'absint' ] ],
    ]);

    // PUT
    register_rest_route( $base, $path, [
        'methods'             => WP_REST_Server::EDITABLE,
        'callback'            => 'rest_update_item',
        'permission_callback' => fn() => current_user_can( 'edit_posts' ),
        'args'                => [ 'id' => [ 'sanitize_callback' => 'absint' ] ],
    ]);

    // DELETE
    register_rest_route( $base, $path, [
        'methods'             => WP_REST_Server::DELETABLE,
        'callback'            => 'rest_delete_item',
        'permission_callback' => fn() => current_user_can( 'delete_posts' ),
        'args'                => [ 'id' => [ 'sanitize_callback' => 'absint' ] ],
    ]);
});
```

---

## Snippet 7 — Require Authentication (Application Passwords / Cookie)

```php
// Option A: Require any logged-in user
'permission_callback' => fn() => is_user_logged_in(),

// Option B: Require specific capability
'permission_callback' => fn() => current_user_can( 'manage_options' ),

// Option C: Require capability + verify nonce (for JS requests from admin)
'permission_callback' => function( WP_REST_Request $request ) {
    $nonce = $request->get_header( 'X-WP-Nonce' );
    if ( ! wp_verify_nonce( $nonce, 'wp_rest' ) ) {
        return new WP_Error( 'invalid_nonce', 'Nonce verification failed.', [ 'status' => 403 ] );
    }
    return current_user_can( 'edit_posts' );
},

// Option D: API Key authentication
'permission_callback' => function( WP_REST_Request $request ) {
    $api_key = sanitize_text_field( $request->get_header( 'X-API-Key' ) ?? '' );
    return hash_equals( get_option( 'my_plugin_api_key' ), $api_key );
},
```

---

## Snippet 8 — Add Custom Field to Existing REST Response (Posts)

```php
add_action( 'rest_api_init', function() {
    // Add a custom field to the post REST response
    register_rest_field( 'post', 'reading_time', [
        'get_callback' => function( array $post_data ): string {
            $content    = get_post_field( 'post_content', $post_data['id'] );
            $word_count = str_word_count( wp_strip_all_tags( $content ) );
            $minutes    = max( 1, (int) ceil( $word_count / 200 ) );
            return "{$minutes} min read";
        },
        'schema' => [
            'description' => 'Estimated reading time',
            'type'        => 'string',
            'context'     => [ 'view' ],
        ],
    ]);
});
```

---

## Snippet 9 — Add Custom Field to User REST Response

```php
add_action( 'rest_api_init', function() {
    register_rest_field( 'user', 'avatar_large', [
        'get_callback' => fn( $user ) => get_avatar_url( $user['id'], [ 'size' => 200 ] ),
        'schema'       => [ 'type' => 'string', 'format' => 'uri' ],
    ]);

    register_rest_field( 'user', 'post_count', [
        'get_callback' => fn( $user ) => (int) count_user_posts( $user['id'] ),
        'schema'       => [ 'type' => 'integer' ],
    ]);
});
```

---

## Snippet 10 — Filter REST API Query (Before Query Runs)

```php
// Modify the WP_Query args before a REST request runs
add_filter( 'rest_post_query', 'modify_rest_post_query', 10, 2 );
function modify_rest_post_query( array $args, WP_REST_Request $request ): array {
    // Only show posts from current user unless admin
    if ( ! current_user_can( 'edit_others_posts' ) ) {
        $args['author'] = get_current_user_id();
    }

    return $args;
}
```

---

## Snippet 11 — Disable REST API for Non-Logged-in Users

```php
add_filter( 'rest_authentication_errors', function( $result ) {
    if ( ! empty( $result ) ) {
        return $result; // Already authenticated or errored
    }

    if ( ! is_user_logged_in() ) {
        return new WP_Error(
            'rest_not_logged_in',
            'You must be logged in to access the REST API.',
            [ 'status' => 401 ]
        );
    }

    return $result;
});
```

---

## Snippet 12 — Remove Sensitive Data from REST Responses

```php
// Remove email from user endpoint for non-admins
add_filter( 'rest_prepare_user', 'sanitize_user_rest_response', 10, 3 );
function sanitize_user_rest_response(
    WP_REST_Response $response,
    WP_User $user,
    WP_REST_Request $request
): WP_REST_Response {
    if ( ! current_user_can( 'list_users' ) ) {
        $data = $response->get_data();
        unset( $data['email'] );
        unset( $data['capabilities'] );
        unset( $data['extra_capabilities'] );
        $response->set_data( $data );
    }
    return $response;
}
```

---

## Snippet 13 — Add Custom REST Endpoint with Query Parameters

```php
add_action( 'rest_api_init', function() {
    register_rest_route( 'my-plugin/v1', '/search', [
        'methods'             => WP_REST_Server::READABLE,
        'callback'            => 'rest_search_posts',
        'permission_callback' => '__return_true',
        'args'                => [
            'q' => [
                'required'          => true,
                'type'              => 'string',
                'sanitize_callback' => 'sanitize_text_field',
                'minLength'         => 2,
            ],
            'per_page' => [
                'type'              => 'integer',
                'default'           => 10,
                'minimum'           => 1,
                'maximum'           => 50,
                'sanitize_callback' => 'absint',
            ],
            'page' => [
                'type'              => 'integer',
                'default'           => 1,
                'minimum'           => 1,
                'sanitize_callback' => 'absint',
            ],
        ],
    ]);
});

function rest_search_posts( WP_REST_Request $request ): WP_REST_Response {
    $query = new WP_Query([
        's'              => $request->get_param( 'q' ),
        'post_type'      => 'post',
        'post_status'    => 'publish',
        'posts_per_page' => $request->get_param( 'per_page' ),
        'paged'          => $request->get_param( 'page' ),
    ]);

    $posts = array_map( fn( $p ) => [
        'id'    => $p->ID,
        'title' => get_the_title( $p ),
        'link'  => get_permalink( $p ),
    ], $query->posts );

    $response = rest_ensure_response( $posts );
    $response->header( 'X-WP-Total',     $query->found_posts );
    $response->header( 'X-WP-TotalPages', $query->max_num_pages );

    return $response;
}
```

---

## Snippet 14 — REST API Response with Pagination Headers

```php
function build_paginated_response( WP_Query $query, array $items ): WP_REST_Response {
    $response = rest_ensure_response( $items );

    $response->header( 'X-WP-Total',      $query->found_posts );
    $response->header( 'X-WP-TotalPages', $query->max_num_pages );

    return $response;
}
```

---

## Snippet 15 — Register REST Meta Field

```php
// Register post meta and expose it in REST API
add_action( 'init', function() {
    register_post_meta( 'post', '_reading_level', [
        'show_in_rest'      => true,
        'single'            => true,
        'type'              => 'string',
        'sanitize_callback' => 'sanitize_text_field',
        'auth_callback'     => fn() => current_user_can( 'edit_posts' ),
    ]);
});
```

---

## Snippet 16 — Fetch WordPress REST API with JavaScript

```javascript
// Fetch posts from the REST API
async function fetchPosts( page = 1, perPage = 10 ) {
    const url = new URL( `${window.location.origin}/wp-json/wp/v2/posts` );
    url.searchParams.set( 'page',     page );
    url.searchParams.set( 'per_page', perPage );
    url.searchParams.set( '_embed',   '1' ); // Include featured image, author

    const response = await fetch( url.toString() );

    const total      = response.headers.get( 'X-WP-Total' );
    const totalPages = response.headers.get( 'X-WP-TotalPages' );
    const posts      = await response.json();

    return { posts, total: parseInt( total ), totalPages: parseInt( totalPages ) };
}

// Authenticated request (requires nonce)
async function createPost( title, content ) {
    const response = await fetch( '/wp-json/wp/v2/posts', {
        method:  'POST',
        headers: {
            'Content-Type': 'application/json',
            'X-WP-Nonce':   wpApiSettings.nonce,
        },
        body: JSON.stringify({ title, content, status: 'publish' }),
    });

    return response.json();
}
```

---

## Snippet 17 — Custom REST Controller Class

```php
class My_REST_Controller extends WP_REST_Controller {

    protected $namespace = 'my-plugin/v1';
    protected $rest_base = 'items';

    public function register_routes(): void {
        register_rest_route( $this->namespace, '/' . $this->rest_base, [
            [
                'methods'             => WP_REST_Server::READABLE,
                'callback'            => [ $this, 'get_items' ],
                'permission_callback' => [ $this, 'get_items_permissions_check' ],
            ],
            [
                'methods'             => WP_REST_Server::CREATABLE,
                'callback'            => [ $this, 'create_item' ],
                'permission_callback' => [ $this, 'create_item_permissions_check' ],
            ],
        ]);
    }

    public function get_items_permissions_check( $request ): bool {
        return true; // Public
    }

    public function create_item_permissions_check( $request ): bool {
        return current_user_can( 'edit_posts' );
    }

    public function get_items( WP_REST_Request $request ): WP_REST_Response {
        return rest_ensure_response( [ 'items' => [] ] );
    }

    public function create_item( WP_REST_Request $request ): WP_REST_Response|WP_Error {
        return rest_ensure_response( [ 'created' => true ] );
    }
}

// Register routes
add_action( 'rest_api_init', function() {
    ( new My_REST_Controller() )->register_routes();
});
```

---

## Snippet 18 — Add CORS Headers for Headless WordPress

```php
add_action( 'rest_api_init', function() {
    remove_filter( 'rest_pre_serve_request', 'rest_send_cors_headers' );

    add_filter( 'rest_pre_serve_request', function( $value ) {
        $allowed_origins = [
            'https://myheadlessfrontend.com',
            'http://localhost:3000',
        ];

        $origin = $_SERVER['HTTP_ORIGIN'] ?? '';

        if ( in_array( $origin, $allowed_origins, true ) ) {
            header( "Access-Control-Allow-Origin: {$origin}" );
            header( 'Access-Control-Allow-Credentials: true' );
            header( 'Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS' );
            header( 'Access-Control-Allow-Headers: Authorization, Content-Type, X-WP-Nonce' );
        }

        return $value;
    });
});
```

---

## Snippet 19 — Cache REST API Responses with Transients

```php
function get_rest_cached( WP_REST_Request $request ): WP_REST_Response {
    $cache_key = 'rest_' . md5( $request->get_route() . serialize( $request->get_params() ) );
    $cached    = get_transient( $cache_key );

    if ( false !== $cached ) {
        $response = rest_ensure_response( $cached );
        $response->header( 'X-Cache', 'HIT' );
        return $response;
    }

    // Do expensive work
    $data = fetch_expensive_data( $request->get_param( 'id' ) );

    set_transient( $cache_key, $data, 15 * MINUTE_IN_SECONDS );

    $response = rest_ensure_response( $data );
    $response->header( 'X-Cache', 'MISS' );
    return $response;
}
```

---

## Snippet 20 — Extend Core Post REST Response

```php
// Add featured image URL directly to post response (avoids _embed)
add_filter( 'rest_prepare_post', 'add_featured_image_to_rest', 10, 3 );
function add_featured_image_to_rest(
    WP_REST_Response $response,
    WP_Post $post,
    WP_REST_Request $request
): WP_REST_Response {
    $data = $response->get_data();

    $data['featured_image_url'] = get_the_post_thumbnail_url( $post->ID, 'large' ) ?: null;
    $data['author_name']        = get_the_author_meta( 'display_name', $post->post_author );
    $data['category_names']     = wp_list_pluck( get_the_category( $post->ID ), 'name' );

    $response->set_data( $data );
    return $response;
}
```

---

## Snippet 21 — REST API Rate Limiting

```php
add_filter( 'rest_pre_dispatch', 'rest_api_rate_limit', 10, 3 );
function rest_api_rate_limit( $result, WP_REST_Server $server, WP_REST_Request $request ) {
    // Only rate-limit unauthenticated requests
    if ( is_user_logged_in() ) {
        return $result;
    }

    $ip            = sanitize_text_field( $_SERVER['REMOTE_ADDR'] ?? '' );
    $transient_key = 'rest_rate_' . md5( $ip );
    $requests      = (int) get_transient( $transient_key );

    if ( $requests >= 60 ) { // 60 requests per minute
        return new WP_Error(
            'rest_rate_limit',
            'Too many requests. Please slow down.',
            [ 'status' => 429 ]
        );
    }

    set_transient( $transient_key, $requests + 1, MINUTE_IN_SECONDS );

    return $result;
}
```

---

## Snippet 22 — Validate Complex Args in REST Endpoint

```php
register_rest_route( 'my-plugin/v1', '/orders', [
    'methods'             => WP_REST_Server::CREATABLE,
    'callback'            => 'create_order',
    'permission_callback' => fn() => is_user_logged_in(),
    'args'                => [
        'email' => [
            'required'          => true,
            'type'              => 'string',
            'format'            => 'email',
            'validate_callback' => fn( $v ) => is_email( $v ),
            'sanitize_callback' => 'sanitize_email',
        ],
        'amount' => [
            'required'          => true,
            'type'              => 'number',
            'minimum'           => 0.01,
            'maximum'           => 9999.99,
            'validate_callback' => fn( $v ) => is_numeric( $v ) && $v > 0,
            'sanitize_callback' => fn( $v ) => round( (float) $v, 2 ),
        ],
        'status' => [
            'type'    => 'string',
            'default' => 'pending',
            'enum'    => [ 'pending', 'processing', 'complete' ],
        ],
    ],
]);
```

---

## Snippet 23 — Expose Custom Post Type in REST API

```php
// When registering a CPT — set show_in_rest to true
register_post_type( 'book', [
    'label'        => 'Books',
    'public'       => true,
    'show_in_rest' => true,          // Enables REST API support
    'rest_base'    => 'books',       // URL: /wp-json/wp/v2/books
    'rest_controller_class' => 'WP_REST_Posts_Controller',
    'supports'     => [ 'title', 'editor', 'thumbnail', 'custom-fields' ],
]);

// For an existing CPT registered elsewhere
add_filter( 'register_post_type_args', function( array $args, string $post_type ): array {
    if ( 'book' === $post_type ) {
        $args['show_in_rest'] = true;
        $args['rest_base']    = 'books';
    }
    return $args;
}, 10, 2 );
```

---

## Snippet 24 — REST API Authentication with Application Passwords

```php
// Application Passwords are built into WordPress 5.6+
// Users generate them at: /wp-admin/profile.php → Application Passwords

// Making an authenticated REST request with Application Password:
// Username: your_wp_username
// Password: xxxx xxxx xxxx xxxx xxxx xxxx (generated app password)
// Encode: base64( 'username:password' )

// PHP — make an authenticated internal REST request
function make_authenticated_rest_request( string $route, array $body = [] ): mixed {
    $request = new WP_REST_Request( 'POST', $route );
    $request->set_body_params( $body );

    // Run as a specific user
    wp_set_current_user( 1 );
    $response = rest_do_request( $request );
    wp_set_current_user( 0 );

    return rest_get_server()->response_to_data( $response, false );
}

// JavaScript — using fetch with Application Passwords
async function fetchWithAuth( endpoint, username, appPassword ) {
    const credentials = btoa( `${username}:${appPassword}` );
    const response = await fetch( `/wp-json/${endpoint}`, {
        headers: {
            'Authorization': `Basic ${credentials}`,
            'Content-Type':  'application/json',
        },
    });
    return response.json();
}
```

---

## Snippet 25 — REST API Cheat Sheet

```php
/**
 * WordPress REST API Cheat Sheet
 *
 * METHOD CONSTANTS
 * WP_REST_Server::READABLE   = 'GET'
 * WP_REST_Server::CREATABLE  = 'POST'
 * WP_REST_Server::EDITABLE   = 'POST, PUT, PATCH'
 * WP_REST_Server::DELETABLE  = 'DELETE'
 * WP_REST_Server::ALLMETHODS = 'GET, POST, PUT, PATCH, DELETE'
 *
 * ARG TYPES
 * 'type' => 'string' | 'integer' | 'number' | 'boolean' | 'array' | 'object'
 *
 * ARG FORMATS (for strings)
 * 'format' => 'email' | 'uri' | 'date-time' | 'ip' | 'uuid'
 *
 * COMMON RESPONSE STATUS CODES
 * 200 OK             — Successful GET, PUT, PATCH
 * 201 Created        — Successful POST
 * 204 No Content     — Successful DELETE
 * 400 Bad Request    — Missing/invalid params
 * 401 Unauthorized   — Not logged in
 * 403 Forbidden      — Logged in but no permission
 * 404 Not Found      — Resource doesn't exist
 * 429 Too Many Reqs  — Rate limit exceeded
 * 500 Server Error   — Something broke
 *
 * BUILT-IN CORE ENDPOINTS
 * GET  /wp/v2/posts
 * GET  /wp/v2/posts/{id}
 * POST /wp/v2/posts
 * GET  /wp/v2/pages
 * GET  /wp/v2/users
 * GET  /wp/v2/categories
 * GET  /wp/v2/tags
 * GET  /wp/v2/media
 * GET  /wp/v2/types
 * GET  /wp/v2/taxonomies
 *
 * USEFUL QUERY PARAMS (core endpoints)
 * ?per_page=10       — Items per page (max 100)
 * ?page=2            — Page number
 * ?search=term       — Search
 * ?orderby=date      — Order by field
 * ?order=asc         — ASC or DESC
 * ?categories=1,2    — Filter by category IDs
 * ?_embed            — Include related data (author, featured image)
 * ?_fields=id,title  — Return only specific fields (performance)
 */
```
