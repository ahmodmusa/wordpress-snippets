# 🔌 Plugin Development

> 25 snippets for building professional WordPress plugins — boilerplate, settings, admin pages, custom tables, and more.

---

## Snippet 1 — Plugin Header Boilerplate

```php
<?php
/**
 * Plugin Name:       My Awesome Plugin
 * Plugin URI:        https://github.com/YOUR_USERNAME/my-plugin
 * Description:       A brief, accurate description of what this plugin does.
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

// Plugin constants
define( 'MY_PLUGIN_VERSION', '1.0.0' );
define( 'MY_PLUGIN_FILE',    __FILE__ );
define( 'MY_PLUGIN_DIR',     plugin_dir_path( __FILE__ ) );
define( 'MY_PLUGIN_URL',     plugin_dir_url( __FILE__ ) );
define( 'MY_PLUGIN_SLUG',    'my-plugin' );
```

---

## Snippet 2 — Activation, Deactivation & Uninstall Hooks

```php
// Activation — runs once when plugin is activated
register_activation_hook( MY_PLUGIN_FILE, 'my_plugin_activate' );
function my_plugin_activate(): void {
    // Check minimum requirements
    if ( version_compare( PHP_VERSION, '8.0', '<' ) ) {
        deactivate_plugins( plugin_basename( MY_PLUGIN_FILE ) );
        wp_die( 'This plugin requires PHP 8.0 or higher.' );
    }

    // Set default options
    add_option( 'my_plugin_version', MY_PLUGIN_VERSION );
    add_option( 'my_plugin_settings', [
        'enabled'   => true,
        'api_key'   => '',
        'log_level' => 'error',
    ]);

    // Create custom DB table, flush rewrite rules, etc.
    flush_rewrite_rules();
}

// Deactivation — runs when plugin is deactivated (NOT uninstalled)
register_deactivation_hook( MY_PLUGIN_FILE, 'my_plugin_deactivate' );
function my_plugin_deactivate(): void {
    wp_clear_scheduled_hook( 'my_plugin_cron_hook' );
    flush_rewrite_rules();
}
```

```php
// uninstall.php — runs when plugin is deleted from admin
// Create this as a SEPARATE FILE: /my-plugin/uninstall.php
if ( ! defined( 'WP_UNINSTALL_PLUGIN' ) ) {
    exit;
}

// Delete all plugin data
delete_option( 'my_plugin_version' );
delete_option( 'my_plugin_settings' );

global $wpdb;
$wpdb->query( "DROP TABLE IF EXISTS {$wpdb->prefix}my_plugin_logs" );
```

---

## Snippet 3 — Singleton Class Pattern

```php
class My_Plugin {

    private static ?My_Plugin $instance = null;

    public static function get_instance(): self {
        if ( null === self::$instance ) {
            self::$instance = new self();
        }
        return self::$instance;
    }

    private function __construct() {
        $this->define_hooks();
    }

    private function define_hooks(): void {
        add_action( 'init',             [ $this, 'init' ] );
        add_action( 'wp_enqueue_scripts', [ $this, 'enqueue_assets' ] );
    }

    public function init(): void {
        load_plugin_textdomain( 'my-plugin', false, MY_PLUGIN_DIR . 'languages' );
    }

    public function enqueue_assets(): void {
        wp_enqueue_style( 'my-plugin', MY_PLUGIN_URL . 'assets/css/frontend.css', [], MY_PLUGIN_VERSION );
    }
}

// Boot the plugin
add_action( 'plugins_loaded', fn() => My_Plugin::get_instance() );
```

---

## Snippet 4 — Add Admin Menu Page

```php
add_action( 'admin_menu', 'my_plugin_add_menu' );
function my_plugin_add_menu(): void {
    // Top-level menu page
    add_menu_page(
        __( 'My Plugin',          'my-plugin' ), // Page title
        __( 'My Plugin',          'my-plugin' ), // Menu title
        'manage_options',                        // Capability
        'my-plugin',                             // Menu slug
        'render_my_plugin_page',                 // Callback
        'dashicons-admin-plugins',               // Icon
        60                                       // Position
    );

    // Submenu pages
    add_submenu_page(
        'my-plugin',
        __( 'Settings', 'my-plugin' ),
        __( 'Settings', 'my-plugin' ),
        'manage_options',
        'my-plugin-settings',
        'render_my_plugin_settings'
    );

    add_submenu_page(
        'my-plugin',
        __( 'Logs', 'my-plugin' ),
        __( 'Logs', 'my-plugin' ),
        'manage_options',
        'my-plugin-logs',
        'render_my_plugin_logs'
    );
}

function render_my_plugin_page(): void {
    if ( ! current_user_can( 'manage_options' ) ) return;
    echo '<div class="wrap"><h1>' . esc_html( get_admin_page_title() ) . '</h1></div>';
}
```

---

## Snippet 5 — Settings API (Full Example)

```php
// Register settings
add_action( 'admin_init', 'my_plugin_register_settings' );
function my_plugin_register_settings(): void {
    register_setting( 'my_plugin_options', 'my_plugin_settings', [
        'sanitize_callback' => 'sanitize_my_plugin_settings',
        'default'           => [],
    ]);

    add_settings_section(
        'my_plugin_main',
        __( 'General Settings', 'my-plugin' ),
        '__return_false',
        'my-plugin-settings'
    );

    add_settings_field(
        'api_key',
        __( 'API Key', 'my-plugin' ),
        'render_api_key_field',
        'my-plugin-settings',
        'my_plugin_main'
    );
}

function render_api_key_field(): void {
    $options = get_option( 'my_plugin_settings', [] );
    $api_key = $options['api_key'] ?? '';
    ?>
    <input type="text"
           name="my_plugin_settings[api_key]"
           value="<?php echo esc_attr( $api_key ); ?>"
           class="regular-text">
    <p class="description"><?php esc_html_e( 'Enter your API key.', 'my-plugin' ); ?></p>
    <?php
}

function sanitize_my_plugin_settings( array $input ): array {
    return [
        'api_key'   => sanitize_text_field( $input['api_key'] ?? '' ),
        'enabled'   => (bool) ( $input['enabled'] ?? false ),
        'log_level' => in_array( $input['log_level'] ?? '', [ 'debug', 'info', 'error' ], true )
                       ? $input['log_level']
                       : 'error',
    ];
}

// Render the settings page
function render_my_plugin_settings(): void {
    if ( ! current_user_can( 'manage_options' ) ) return;
    ?>
    <div class="wrap">
        <h1><?php echo esc_html( get_admin_page_title() ); ?></h1>
        <?php settings_errors( 'my_plugin_options' ); ?>
        <form method="post" action="options.php">
            <?php
            settings_fields( 'my_plugin_options' );
            do_settings_sections( 'my-plugin-settings' );
            submit_button();
            ?>
        </form>
    </div>
    <?php
}
```

---

## Snippet 6 — Create Custom Database Table

```php
register_activation_hook( MY_PLUGIN_FILE, 'create_my_plugin_tables' );
function create_my_plugin_tables(): void {
    global $wpdb;

    $charset_collate = $wpdb->get_charset_collate();
    $table           = $wpdb->prefix . 'my_plugin_logs';

    $sql = "CREATE TABLE IF NOT EXISTS {$table} (
        id          bigint(20)   NOT NULL AUTO_INCREMENT,
        user_id     bigint(20)   NOT NULL DEFAULT 0,
        action      varchar(100) NOT NULL DEFAULT '',
        message     text         NOT NULL,
        level       varchar(20)  NOT NULL DEFAULT 'info',
        created_at  datetime     NOT NULL DEFAULT CURRENT_TIMESTAMP,
        PRIMARY KEY (id),
        KEY user_id (user_id),
        KEY level (level),
        KEY created_at (created_at)
    ) {$charset_collate};";

    require_once ABSPATH . 'wp-admin/includes/upgrade.php';
    dbDelta( $sql ); // Safe for CREATE and ALTER

    update_option( 'my_plugin_db_version', '1.0' );
}
```

---

## Snippet 7 — Handle Database Migrations

```php
add_action( 'plugins_loaded', 'maybe_upgrade_my_plugin_db' );
function maybe_upgrade_my_plugin_db(): void {
    $current_version = get_option( 'my_plugin_db_version', '0' );

    if ( version_compare( $current_version, '1.1', '<' ) ) {
        global $wpdb;
        $table = $wpdb->prefix . 'my_plugin_logs';

        // Add new column
        $wpdb->query( "ALTER TABLE {$table} ADD COLUMN ip_address varchar(45) NOT NULL DEFAULT '' AFTER message" );
        update_option( 'my_plugin_db_version', '1.1' );
    }

    if ( version_compare( $current_version, '1.2', '<' ) ) {
        // Another migration...
        update_option( 'my_plugin_db_version', '1.2' );
    }
}
```

---

## Snippet 8 — Add Shortcode

```php
add_action( 'init', 'register_my_shortcodes' );
function register_my_shortcodes(): void {
    add_shortcode( 'my_button', 'render_my_button_shortcode' );
}

function render_my_button_shortcode( array $atts, string $content = '' ): string {
    $atts = shortcode_atts([
        'url'    => '#',
        'style'  => 'primary',
        'target' => '_self',
        'class'  => '',
    ], $atts, 'my_button' );

    $url     = esc_url( $atts['url'] );
    $style   = sanitize_html_class( $atts['style'] );
    $target  = in_array( $atts['target'], [ '_self', '_blank' ], true ) ? $atts['target'] : '_self';
    $classes = 'btn btn-' . $style . ( $atts['class'] ? ' ' . sanitize_html_class( $atts['class'] ) : '' );
    $content = $content ? wp_kses_post( $content ) : __( 'Click Here', 'my-plugin' );
    $rel     = '_blank' === $target ? ' rel="noopener noreferrer"' : '';

    return "<a href=\"{$url}\" class=\"{$classes}\" target=\"{$target}\"{$rel}>{$content}</a>";
}

// Usage: [my_button url="https://example.com" style="primary" target="_blank"]Learn More[/my_button]
```

---

## Snippet 9 — Add Admin Notice

```php
// Persistent admin notice (stored in option)
add_action( 'admin_notices', 'my_plugin_admin_notices' );
function my_plugin_admin_notices(): void {
    $settings = get_option( 'my_plugin_settings', [] );

    // Show warning if API key is missing
    if ( empty( $settings['api_key'] ) ) {
        $settings_url = admin_url( 'admin.php?page=my-plugin-settings' );
        ?>
        <div class="notice notice-warning is-dismissible">
            <p>
                <?php printf(
                    /* translators: %s: settings page URL */
                    wp_kses( __( 'My Plugin: Please <a href="%s">add your API key</a> to get started.', 'my-plugin' ), [ 'a' => [ 'href' => [] ] ] ),
                    esc_url( $settings_url )
                ); ?>
            </p>
        </div>
        <?php
    }
}

// One-time notice after activation
add_action( 'admin_notices', 'my_plugin_activation_notice' );
function my_plugin_activation_notice(): void {
    if ( ! get_transient( 'my_plugin_activated' ) ) return;

    delete_transient( 'my_plugin_activated' );
    ?>
    <div class="notice notice-success is-dismissible">
        <p><?php esc_html_e( 'My Plugin activated! Configure it in the settings.', 'my-plugin' ); ?></p>
    </div>
    <?php
}

// Set the transient on activation
register_activation_hook( MY_PLUGIN_FILE, function() {
    set_transient( 'my_plugin_activated', true, 30 );
});
```

---

## Snippet 10 — Add Plugin Action Links

```php
// Add links next to "Activate/Deactivate" on plugins list page
add_filter( 'plugin_action_links_' . plugin_basename( MY_PLUGIN_FILE ), 'my_plugin_action_links' );
function my_plugin_action_links( array $links ): array {
    $custom_links = [
        '<a href="' . esc_url( admin_url( 'admin.php?page=my-plugin-settings' ) ) . '">'
            . esc_html__( 'Settings', 'my-plugin' ) . '</a>',
        '<a href="https://yoursite.com/docs" target="_blank">'
            . esc_html__( 'Docs', 'my-plugin' ) . '</a>',
    ];

    return array_merge( $custom_links, $links );
}

// Add links in the plugin meta row (below the description)
add_filter( 'plugin_row_meta', 'my_plugin_row_meta', 10, 2 );
function my_plugin_row_meta( array $links, string $file ): array {
    if ( plugin_basename( MY_PLUGIN_FILE ) !== $file ) {
        return $links;
    }

    $links[] = '<a href="https://yoursite.com/support" target="_blank">'
        . esc_html__( 'Support', 'my-plugin' ) . '</a>';

    return $links;
}
```

---

## Snippet 11 — Schedule a Cron Job

```php
// Register activation
register_activation_hook( MY_PLUGIN_FILE, 'my_plugin_schedule_cron' );
function my_plugin_schedule_cron(): void {
    if ( ! wp_next_scheduled( 'my_plugin_daily_task' ) ) {
        wp_schedule_event( time(), 'daily', 'my_plugin_daily_task' );
    }
}

// Clear on deactivation
register_deactivation_hook( MY_PLUGIN_FILE, function() {
    wp_clear_scheduled_hook( 'my_plugin_daily_task' );
});

// The actual task
add_action( 'my_plugin_daily_task', 'run_my_plugin_daily_task' );
function run_my_plugin_daily_task(): void {
    // Clean up logs older than 30 days
    global $wpdb;
    $table = $wpdb->prefix . 'my_plugin_logs';
    $wpdb->query(
        $wpdb->prepare(
            "DELETE FROM {$table} WHERE created_at < %s",
            gmdate( 'Y-m-d H:i:s', strtotime( '-30 days' ) )
        )
    );
}
```

---

## Snippet 12 — WP_List_Table for Admin Data

```php
if ( ! class_exists( 'WP_List_Table' ) ) {
    require_once ABSPATH . 'wp-admin/includes/class-wp-list-table.php';
}

class My_Plugin_Logs_Table extends WP_List_Table {

    public function __construct() {
        parent::__construct([
            'singular' => 'log',
            'plural'   => 'logs',
            'ajax'     => false,
        ]);
    }

    public function get_columns(): array {
        return [
            'cb'         => '<input type="checkbox">',
            'message'    => __( 'Message',    'my-plugin' ),
            'level'      => __( 'Level',      'my-plugin' ),
            'created_at' => __( 'Date',       'my-plugin' ),
        ];
    }

    public function prepare_items(): void {
        global $wpdb;
        $table      = $wpdb->prefix . 'my_plugin_logs';
        $per_page   = 20;
        $current    = $this->get_pagenum();
        $total      = (int) $wpdb->get_var( "SELECT COUNT(*) FROM {$table}" );

        $this->items = $wpdb->get_results(
            $wpdb->prepare(
                "SELECT * FROM {$table} ORDER BY created_at DESC LIMIT %d OFFSET %d",
                $per_page,
                ( $current - 1 ) * $per_page
            ),
            ARRAY_A
        );

        $this->set_pagination_args([
            'total_items' => $total,
            'per_page'    => $per_page,
        ]);
    }

    public function column_default( $item, string $column_name ): string {
        return esc_html( $item[ $column_name ] ?? '' );
    }
}
```

---

## Snippet 13 — Options API Helpers

```php
// Get a single nested option value safely
function my_plugin_get_option( string $key, $default = null ) {
    $options = get_option( 'my_plugin_settings', [] );
    return $options[ $key ] ?? $default;
}

// Update a single nested option value
function my_plugin_update_option( string $key, $value ): void {
    $options         = get_option( 'my_plugin_settings', [] );
    $options[ $key ] = $value;
    update_option( 'my_plugin_settings', $options );
}

// Usage
$api_key = my_plugin_get_option( 'api_key', '' );
my_plugin_update_option( 'api_key', 'new-key-123' );
```

---

## Snippet 14 — Custom Capabilities & Roles

```php
// Add custom capability to administrator on activation
register_activation_hook( MY_PLUGIN_FILE, 'add_my_plugin_capabilities' );
function add_my_plugin_capabilities(): void {
    $admin = get_role( 'administrator' );
    if ( $admin ) {
        $admin->add_cap( 'manage_my_plugin' );
        $admin->add_cap( 'view_my_plugin_logs' );
    }
}

// Remove on uninstall (in uninstall.php)
function remove_my_plugin_capabilities(): void {
    foreach ( wp_roles()->roles as $role_name => $role_info ) {
        $role = get_role( $role_name );
        if ( $role ) {
            $role->remove_cap( 'manage_my_plugin' );
            $role->remove_cap( 'view_my_plugin_logs' );
        }
    }
}

// Use in permission checks
if ( current_user_can( 'manage_my_plugin' ) ) {
    // Show plugin admin features
}
```

---

## Snippet 15 — Email Notification Helper

```php
/**
 * Send a templated email notification.
 *
 * @param string $to      Recipient email.
 * @param string $subject Email subject.
 * @param array  $data    Template variables.
 * @return bool
 */
function my_plugin_send_email( string $to, string $subject, array $data = [] ): bool {
    if ( ! is_email( $to ) ) return false;

    $site_name = get_bloginfo( 'name' );
    $headers   = [
        'Content-Type: text/html; charset=UTF-8',
        'From: ' . $site_name . ' <' . get_option( 'admin_email' ) . '>',
    ];

    ob_start();
    // Load email template
    $template = MY_PLUGIN_DIR . 'templates/email.php';
    if ( file_exists( $template ) ) {
        extract( $data, EXTR_SKIP ); // phpcs:ignore
        include $template;
    }
    $message = ob_get_clean();

    add_filter( 'wp_mail_content_type', fn() => 'text/html' );
    $sent = wp_mail( $to, $subject, $message, $headers );
    remove_filter( 'wp_mail_content_type', fn() => 'text/html' );

    return $sent;
}
```

---

## Snippet 16 — Simple Logger Class

```php
class My_Plugin_Logger {

    private static string $table = '';

    public static function init(): void {
        global $wpdb;
        self::$table = $wpdb->prefix . 'my_plugin_logs';
    }

    public static function log( string $message, string $level = 'info', int $user_id = 0 ): void {
        global $wpdb;

        if ( empty( self::$table ) ) self::init();

        $wpdb->insert(
            self::$table,
            [
                'user_id'    => $user_id ?: get_current_user_id(),
                'message'    => $message,
                'level'      => sanitize_key( $level ),
                'created_at' => current_time( 'mysql' ),
            ],
            [ '%d', '%s', '%s', '%s' ]
        );
    }

    public static function error( string $message ): void {
        self::log( $message, 'error' );
    }

    public static function info( string $message ): void {
        self::log( $message, 'info' );
    }

    public static function debug( string $message ): void {
        if ( defined( 'WP_DEBUG' ) && WP_DEBUG ) {
            self::log( $message, 'debug' );
        }
    }
}

// Usage
My_Plugin_Logger::error( 'API request failed.' );
My_Plugin_Logger::info( 'User imported 50 records.' );
```

---

## Snippet 17 — Handle Plugin Update Routines

```php
add_action( 'plugins_loaded', 'my_plugin_maybe_update' );
function my_plugin_maybe_update(): void {
    $installed = get_option( 'my_plugin_version', '0.0.0' );

    if ( version_compare( $installed, MY_PLUGIN_VERSION, '>=' ) ) {
        return; // Already up to date
    }

    // Run migrations based on version
    if ( version_compare( $installed, '1.1.0', '<' ) ) {
        // Migrate from 1.0.x to 1.1.0
        my_plugin_migrate_to_110();
    }

    if ( version_compare( $installed, '1.2.0', '<' ) ) {
        my_plugin_migrate_to_120();
    }

    // Update stored version
    update_option( 'my_plugin_version', MY_PLUGIN_VERSION );
}

function my_plugin_migrate_to_110(): void {
    // Rename old option key
    $old = get_option( 'my_plugin_old_key' );
    if ( $old !== false ) {
        update_option( 'my_plugin_new_key', $old );
        delete_option( 'my_plugin_old_key' );
    }
}
```

---

## Snippet 18 — REST API Integration in Plugin

```php
class My_Plugin_REST_API {

    public function register(): void {
        add_action( 'rest_api_init', [ $this, 'register_routes' ] );
    }

    public function register_routes(): void {
        register_rest_route( 'my-plugin/v1', '/status', [
            'methods'             => WP_REST_Server::READABLE,
            'callback'            => [ $this, 'get_status' ],
            'permission_callback' => fn() => current_user_can( 'manage_my_plugin' ),
        ]);
    }

    public function get_status( WP_REST_Request $request ): WP_REST_Response {
        return rest_ensure_response([
            'version'    => MY_PLUGIN_VERSION,
            'active'     => true,
            'settings'   => [
                'api_key_set' => ! empty( my_plugin_get_option( 'api_key' ) ),
            ],
        ]);
    }
}

( new My_Plugin_REST_API() )->register();
```

---

## Snippet 19 — Add Custom Dashboard Widget

```php
add_action( 'wp_dashboard_setup', 'my_plugin_dashboard_widget' );
function my_plugin_dashboard_widget(): void {
    wp_add_dashboard_widget(
        'my_plugin_stats',
        __( 'My Plugin Stats', 'my-plugin' ),
        'render_my_plugin_dashboard_widget',
        null,
        null,
        'normal',
        'high'
    );
}

function render_my_plugin_dashboard_widget(): void {
    if ( ! current_user_can( 'manage_my_plugin' ) ) {
        echo '<p>' . esc_html__( 'No permission.', 'my-plugin' ) . '</p>';
        return;
    }

    global $wpdb;
    $table = $wpdb->prefix . 'my_plugin_logs';
    $count = (int) $wpdb->get_var( "SELECT COUNT(*) FROM {$table} WHERE level = 'error'" );
    ?>
    <ul>
        <li><?php printf( esc_html__( 'Errors (last 30 days): %d', 'my-plugin' ), $count ); ?></li>
        <li><?php printf( esc_html__( 'Plugin version: %s', 'my-plugin' ), esc_html( MY_PLUGIN_VERSION ) ); ?></li>
    </ul>
    <p><a href="<?php echo esc_url( admin_url( 'admin.php?page=my-plugin-logs' ) ); ?>">
        <?php esc_html_e( 'View all logs →', 'my-plugin' ); ?>
    </a></p>
    <?php
}
```

---

## Snippet 20 — Dependency Check (Require Another Plugin)

```php
add_action( 'admin_init', 'my_plugin_check_dependencies' );
function my_plugin_check_dependencies(): void {
    // Check if WooCommerce is active
    if ( ! class_exists( 'WooCommerce' ) ) {
        add_action( 'admin_notices', function() {
            ?>
            <div class="notice notice-error">
                <p><?php esc_html_e( 'My Plugin requires WooCommerce to be installed and active.', 'my-plugin' ); ?></p>
            </div>
            <?php
        });

        deactivate_plugins( plugin_basename( MY_PLUGIN_FILE ) );
    }
}

// Check for a specific WooCommerce version
function my_plugin_woo_version_check( string $min_version = '7.0' ): bool {
    return defined( 'WC_VERSION' ) && version_compare( WC_VERSION, $min_version, '>=' );
}
```

---

## Snippet 21 — AJAX in Admin Page

```php
// Enqueue admin script with nonce
add_action( 'admin_enqueue_scripts', 'my_plugin_admin_scripts' );
function my_plugin_admin_scripts( string $hook ): void {
    if ( 'my-plugin_page_my-plugin-settings' !== $hook ) return;

    wp_enqueue_script(
        'my-plugin-admin',
        MY_PLUGIN_URL . 'assets/js/admin.js',
        [ 'jquery' ],
        MY_PLUGIN_VERSION,
        true
    );

    wp_localize_script( 'my-plugin-admin', 'MyPluginAdmin', [
        'ajax_url' => admin_url( 'admin-ajax.php' ),
        'nonce'    => wp_create_nonce( 'my_plugin_admin_nonce' ),
        'i18n'     => [
            'saving'  => __( 'Saving...', 'my-plugin' ),
            'saved'   => __( 'Saved!',    'my-plugin' ),
            'error'   => __( 'Error. Please try again.', 'my-plugin' ),
        ],
    ]);
}

// AJAX handler
add_action( 'wp_ajax_my_plugin_test_api', 'my_plugin_test_api_connection' );
function my_plugin_test_api_connection(): void {
    check_ajax_referer( 'my_plugin_admin_nonce', 'nonce' );

    if ( ! current_user_can( 'manage_options' ) ) {
        wp_send_json_error( [ 'message' => 'Permission denied.' ], 403 );
    }

    $api_key  = my_plugin_get_option( 'api_key', '' );
    $response = wp_remote_get( 'https://api.example.com/verify', [
        'headers' => [ 'Authorization' => 'Bearer ' . $api_key ],
        'timeout' => 10,
    ]);

    if ( is_wp_error( $response ) || 200 !== wp_remote_retrieve_response_code( $response ) ) {
        wp_send_json_error( [ 'message' => 'API connection failed.' ] );
    }

    wp_send_json_success( [ 'message' => 'API connection successful!' ] );
}
```

---

## Snippet 22 — Autoloader (PSR-4 Style)

```php
// Simple PSR-4 autoloader — add to main plugin file
spl_autoload_register( function( string $class_name ): void {
    // Only handle our namespace
    $prefix   = 'My_Plugin\\';
    $base_dir = MY_PLUGIN_DIR . 'includes/';

    if ( strncmp( $prefix, $class_name, strlen( $prefix ) ) !== 0 ) {
        return;
    }

    $relative_class = substr( $class_name, strlen( $prefix ) );
    $file           = $base_dir . str_replace( '\\', '/', $relative_class ) . '.php';

    if ( file_exists( $file ) ) {
        require_once $file;
    }
});

// Now classes like My_Plugin\Admin\Settings auto-load from includes/Admin/Settings.php
```

---

## Snippet 23 — User Meta Helper

```php
// Helpers for storing plugin-specific user data
function my_plugin_get_user_data( int $user_id, string $key, $default = null ) {
    $data = get_user_meta( $user_id, 'my_plugin_data', true );
    $data = is_array( $data ) ? $data : [];
    return $data[ $key ] ?? $default;
}

function my_plugin_update_user_data( int $user_id, string $key, $value ): void {
    $data         = get_user_meta( $user_id, 'my_plugin_data', true );
    $data         = is_array( $data ) ? $data : [];
    $data[ $key ] = $value;
    update_user_meta( $user_id, 'my_plugin_data', $data );
}

function my_plugin_delete_user_data( int $user_id, string $key ): void {
    $data = get_user_meta( $user_id, 'my_plugin_data', true );
    $data = is_array( $data ) ? $data : [];
    unset( $data[ $key ] );
    update_user_meta( $user_id, 'my_plugin_data', $data );
}

// Clean up when user is deleted
add_action( 'delete_user', fn( $user_id ) => delete_user_meta( $user_id, 'my_plugin_data' ) );
```

---

## Snippet 24 — Transient-Based API Cache

```php
class My_Plugin_API {

    private string $base_url = 'https://api.example.com/v1/';
    private int    $cache_ttl = 900; // 15 minutes

    public function get( string $endpoint, array $params = [] ): array|WP_Error {
        $cache_key = 'my_plugin_api_' . md5( $endpoint . serialize( $params ) );
        $cached    = get_transient( $cache_key );

        if ( false !== $cached ) {
            return $cached;
        }

        $url      = add_query_arg( $params, $this->base_url . ltrim( $endpoint, '/' ) );
        $api_key  = my_plugin_get_option( 'api_key', '' );

        $response = wp_remote_get( $url, [
            'timeout' => 15,
            'headers' => [ 'Authorization' => 'Bearer ' . $api_key ],
        ]);

        if ( is_wp_error( $response ) ) {
            return $response;
        }

        $code = wp_remote_retrieve_response_code( $response );
        if ( 200 !== $code ) {
            return new WP_Error( 'api_error', "API returned status {$code}." );
        }

        $data = json_decode( wp_remote_retrieve_body( $response ), true );
        set_transient( $cache_key, $data, $this->cache_ttl );

        return $data;
    }
}
```

---

## Snippet 25 — Plugin Development Checklist

```php
/**
 * WordPress Plugin Development Checklist
 *
 * STRUCTURE
 * ✅ Main plugin file has correct plugin header comment
 * ✅ Unique prefix used on ALL functions, classes, constants, options
 * ✅ ABSPATH check at top of every PHP file
 * ✅ uninstall.php created to clean up all plugin data on delete
 * ✅ Text domain matches plugin slug
 * ✅ languages/ folder exists with .pot file
 *
 * SECURITY
 * ✅ All $_POST/$_GET data sanitized
 * ✅ All output escaped with esc_html(), esc_attr(), esc_url()
 * ✅ Nonces verified on all forms and AJAX requests
 * ✅ current_user_can() checked before every privileged action
 * ✅ $wpdb->prepare() used for all custom queries
 * ✅ Settings sanitized with sanitize_callback in register_setting()
 *
 * HOOKS & STANDARDS
 * ✅ Hooks added inside functions (never at file root)
 * ✅ Plugin loaded on plugins_loaded (not init)
 * ✅ No wp_redirect() — use wp_safe_redirect()
 * ✅ No exit/die — use wp_die() with a message
 * ✅ No hardcoded URLs — use plugins_url(), admin_url(), etc.
 * ✅ Translation functions used: __(), _e(), esc_html__()
 *
 * ACTIVATION / DEACTIVATION
 * ✅ register_activation_hook() used (not 'init')
 * ✅ register_deactivation_hook() clears cron jobs
 * ✅ Upgrade routine handles version migrations
 * ✅ Custom DB tables use dbDelta() (not raw CREATE TABLE)
 *
 * RELEASE
 * ✅ Tested on latest WordPress version
 * ✅ Tested on PHP 8.0, 8.1, 8.2
 * ✅ No PHP warnings or notices (WP_DEBUG = true)
 * ✅ readme.txt follows WordPress.org format
 * ✅ Plugin tested with Query Monitor active
 */
```
