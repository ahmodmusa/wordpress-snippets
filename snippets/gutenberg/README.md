# 🧱 Gutenberg & Block Editor

> 25 snippets for working with the WordPress block editor — blocks, patterns, variations, and Full Site Editing.

---

## Snippet 1 — Enqueue Block Editor Assets (Editor Only)

```php
add_action( 'enqueue_block_editor_assets', 'my_block_editor_assets' );
function my_block_editor_assets(): void {
    wp_enqueue_script(
        'my-block-editor',
        plugins_url( 'build/index.js', __FILE__ ),
        [ 'wp-blocks', 'wp-element', 'wp-editor', 'wp-components', 'wp-i18n' ],
        filemtime( plugin_dir_path( __FILE__ ) . 'build/index.js' )
    );

    wp_enqueue_style(
        'my-block-editor-style',
        plugins_url( 'build/editor.css', __FILE__ ),
        [ 'wp-edit-blocks' ],
        filemtime( plugin_dir_path( __FILE__ ) . 'build/editor.css' )
    );
}
```

---

## Snippet 2 — Enqueue Block Frontend & Editor Styles

```php
add_action( 'enqueue_block_assets', 'my_block_assets' );
function my_block_assets(): void {
    // Loads on both frontend AND in the editor
    wp_enqueue_style(
        'my-block-style',
        plugins_url( 'build/style.css', __FILE__ ),
        [],
        filemtime( plugin_dir_path( __FILE__ ) . 'build/style.css' )
    );
}
```

---

## Snippet 3 — Register a Custom Block Category

```php
add_filter( 'block_categories_all', 'register_my_block_category', 10, 2 );
function register_my_block_category( array $categories, WP_Block_Editor_Context $context ): array {
    return array_merge(
        [
            [
                'slug'  => 'my-plugin',
                'title' => __( 'My Plugin Blocks', 'my-plugin' ),
                'icon'  => 'layout',
            ],
        ],
        $categories
    );
}
```

---

## Snippet 4 — Register a Static Block (block.json)

```json
{
    "$schema": "https://schemas.wp.org/trunk/block.json",
    "apiVersion": 3,
    "name": "my-plugin/notice",
    "version": "1.0.0",
    "title": "Notice",
    "category": "my-plugin",
    "description": "Display a notice box with a message.",
    "supports": {
        "html": false,
        "color": { "background": true, "text": true },
        "spacing": { "padding": true, "margin": true }
    },
    "attributes": {
        "message": { "type": "string", "default": "" },
        "type":    { "type": "string", "default": "info" }
    },
    "editorScript": "file:./index.js",
    "editorStyle":  "file:./editor.css",
    "style":        "file:./style.css",
    "render":       "file:./render.php"
}
```

---

## Snippet 5 — Register Block from PHP (Legacy / Simple)

```php
add_action( 'init', 'register_my_blocks' );
function register_my_blocks(): void {
    // If using block.json — just point to the folder
    register_block_type( plugin_dir_path( __FILE__ ) . 'blocks/notice/' );

    // Or register fully from PHP
    register_block_type( 'my-plugin/alert', [
        'editor_script'   => 'my-alert-block',
        'render_callback' => 'render_alert_block',
        'attributes'      => [
            'message' => [ 'type' => 'string', 'default' => '' ],
            'style'   => [ 'type' => 'string', 'default' => 'info' ],
        ],
    ]);
}

function render_alert_block( array $attributes, string $content ): string {
    $message = esc_html( $attributes['message'] ?? '' );
    $style   = sanitize_html_class( $attributes['style'] ?? 'info' );

    return "<div class=\"wp-block-alert alert-{$style}\"><p>{$message}</p></div>";
}
```

---

## Snippet 6 — Add Custom Block Pattern

```php
add_action( 'init', 'register_my_block_patterns' );
function register_my_block_patterns(): void {
    register_block_pattern(
        'my-plugin/hero-section',
        [
            'title'       => __( 'Hero Section', 'my-plugin' ),
            'description' => __( 'Full-width hero with heading and CTA button', 'my-plugin' ),
            'categories'  => [ 'featured' ],
            'keywords'    => [ 'hero', 'banner', 'header' ],
            'content'     => '<!-- wp:group {"align":"full","style":{"spacing":{"padding":{"top":"80px","bottom":"80px"}}}} -->
<div class="wp-block-group alignfull">
<!-- wp:heading {"textAlign":"center","level":1} -->
<h1 class="has-text-align-center">Welcome to Our Site</h1>
<!-- /wp:heading -->
<!-- wp:paragraph {"align":"center"} -->
<p class="has-text-align-center">A short tagline that explains what you do.</p>
<!-- /wp:paragraph -->
<!-- wp:buttons {"layout":{"type":"flex","justifyContent":"center"}} -->
<div class="wp-block-buttons">
<!-- wp:button -->
<div class="wp-block-button"><a class="wp-block-button__link wp-element-button">Get Started</a></div>
<!-- /wp:button -->
</div>
<!-- /wp:buttons -->
</div>
<!-- /wp:group -->',
        ]
    );
}
```

---

## Snippet 7 — Register Block Pattern Category

```php
add_action( 'init', 'register_my_pattern_categories' );
function register_my_pattern_categories(): void {
    register_block_pattern_category(
        'my-plugin',
        [ 'label' => __( 'My Plugin', 'my-plugin' ) ]
    );
}
```

---

## Snippet 8 — Load Patterns from PHP Files (WordPress 6.0+)

```php
// Place pattern files in /patterns/ folder in your theme or plugin
// Each file is auto-registered — no PHP registration needed

// Example: /patterns/hero.php
<?php
/**
 * Title: Hero Section
 * Slug: my-theme/hero
 * Categories: featured
 * Keywords: hero, banner
 * Block Types: core/template-part/header
 */
?>
<!-- wp:group {"align":"full"} -->
<div class="wp-block-group alignfull">
<!-- wp:heading -->
<h1>Hello World</h1>
<!-- /wp:heading -->
</div>
<!-- /wp:group -->
```

---

## Snippet 9 — Add theme.json (Full Site Editing)

```json
{
    "$schema": "https://schemas.wp.org/trunk/theme.json",
    "version": 2,
    "settings": {
        "color": {
            "palette": [
                { "slug": "primary",   "color": "#0073aa", "name": "Primary" },
                { "slug": "secondary", "color": "#005177", "name": "Secondary" },
                { "slug": "white",     "color": "#ffffff", "name": "White" },
                { "slug": "black",     "color": "#000000", "name": "Black" }
            ]
        },
        "typography": {
            "fontSizes": [
                { "slug": "small",       "size": "14px",  "name": "Small" },
                { "slug": "medium",      "size": "18px",  "name": "Medium" },
                { "slug": "large",       "size": "24px",  "name": "Large" },
                { "slug": "x-large",     "size": "36px",  "name": "X-Large" },
                { "slug": "xx-large",    "size": "48px",  "name": "XX-Large" }
            ]
        },
        "layout": {
            "contentSize": "800px",
            "wideSize": "1200px"
        },
        "spacing": {
            "spacingScale": {
                "steps": 7
            }
        }
    },
    "styles": {
        "color": {
            "background": "var(--wp--preset--color--white)",
            "text": "var(--wp--preset--color--black)"
        },
        "typography": {
            "fontSize": "var(--wp--preset--font-size--medium)",
            "lineHeight": "1.6"
        }
    }
}
```

---

## Snippet 10 — Disable Specific Blocks

```php
add_filter( 'allowed_block_types_all', 'restrict_allowed_blocks', 10, 2 );
function restrict_allowed_blocks( $allowed_blocks, WP_Block_Editor_Context $context ): array {
    // Return true to allow all, or an array of allowed block names
    $allowed = [
        'core/paragraph',
        'core/heading',
        'core/image',
        'core/list',
        'core/quote',
        'core/button',
        'core/buttons',
        'core/group',
        'core/columns',
        'core/column',
        'core/separator',
        'core/spacer',
    ];

    return $allowed;
}
```

---

## Snippet 11 — Add Custom Block Variation

```javascript
// variations.js — register a variation of core/button
import { registerBlockVariation } from '@wordpress/blocks';

registerBlockVariation( 'core/button', {
    name:        'my-plugin/cta-button',
    title:       'CTA Button',
    description: 'A styled call-to-action button',
    isDefault:   false,
    attributes: {
        className: 'is-style-cta',
        text:      'Get Started',
    },
    icon: 'megaphone',
    scope: [ 'inserter' ],
} );
```

---

## Snippet 12 — Add Custom Block Style Variation

```php
// PHP — register a style variation for core/button
add_action( 'init', 'register_my_block_styles' );
function register_my_block_styles(): void {
    register_block_style( 'core/button', [
        'name'  => 'outline-white',
        'label' => __( 'Outline White', 'my-theme' ),
    ]);

    register_block_style( 'core/quote', [
        'name'  => 'large-quote',
        'label' => __( 'Large Quote', 'my-theme' ),
    ]);
}

// Unregister a style you don't want
add_action( 'init', 'unregister_unused_block_styles' );
function unregister_unused_block_styles(): void {
    unregister_block_style( 'core/quote', 'large' );
    unregister_block_style( 'core/pullquote', 'solid-color' );
}
```

---

## Snippet 13 — Add Custom Toolbar Button to a Block

```javascript
// editor.js — add a custom toolbar button
import { addFilter } from '@wordpress/hooks';
import { createHigherOrderComponent } from '@wordpress/compose';
import { BlockControls } from '@wordpress/block-editor';
import { ToolbarGroup, ToolbarButton } from '@wordpress/components';

const withCustomToolbar = createHigherOrderComponent( ( BlockEdit ) => {
    return ( props ) => {
        if ( props.name !== 'core/paragraph' ) {
            return <BlockEdit { ...props } />;
        }

        return (
            <>
                <BlockControls>
                    <ToolbarGroup>
                        <ToolbarButton
                            icon="star-filled"
                            label="Mark as Featured"
                            onClick={ () => props.setAttributes({ className: 'is-featured' }) }
                        />
                    </ToolbarGroup>
                </BlockControls>
                <BlockEdit { ...props } />
            </>
        );
    };
}, 'withCustomToolbar' );

addFilter( 'editor.BlockEdit', 'my-plugin/custom-toolbar', withCustomToolbar );
```

---

## Snippet 14 — Add Custom Inspector Controls (Sidebar Panel)

```javascript
// editor.js — add a custom panel to the block inspector sidebar
import { addFilter } from '@wordpress/hooks';
import { createHigherOrderComponent } from '@wordpress/compose';
import { InspectorControls } from '@wordpress/block-editor';
import { PanelBody, ToggleControl } from '@wordpress/components';

const withInspectorControls = createHigherOrderComponent( ( BlockEdit ) => {
    return ( props ) => {
        if ( props.name !== 'core/paragraph' ) {
            return <BlockEdit { ...props } />;
        }

        const { attributes, setAttributes } = props;

        return (
            <>
                <InspectorControls>
                    <PanelBody title="Custom Settings" initialOpen={ true }>
                        <ToggleControl
                            label="Show as Featured"
                            checked={ attributes.isFeatured || false }
                            onChange={ ( val ) => setAttributes({ isFeatured: val }) }
                        />
                    </PanelBody>
                </InspectorControls>
                <BlockEdit { ...props } />
            </>
        );
    };
}, 'withInspectorControls' );

addFilter( 'editor.BlockEdit', 'my-plugin/inspector-controls', withInspectorControls );
```

---

## Snippet 15 — Server-Side Render Block (render.php)

```php
<?php
// render.php — used when block.json has "render": "file:./render.php"
// $attributes, $content, and $block are available automatically

$type    = sanitize_html_class( $attributes['type'] ?? 'info' );
$message = wp_kses_post( $attributes['message'] ?? '' );
$wrapper = get_block_wrapper_attributes( [ 'class' => "notice notice-{$type}" ] );
?>
<div <?php echo $wrapper; // phpcs:ignore ?>>
    <p><?php echo $message; // phpcs:ignore ?></p>
</div>
```

---

## Snippet 16 — Add Block Supports (align, spacing, color)

```json
{
    "name": "my-plugin/card",
    "supports": {
        "align":  [ "wide", "full" ],
        "anchor": true,
        "color": {
            "background":    true,
            "text":          true,
            "gradients":     true,
            "link":          true
        },
        "spacing": {
            "padding":  true,
            "margin":   true,
            "blockGap": true
        },
        "typography": {
            "fontSize":   true,
            "lineHeight": true
        },
        "dimensions": {
            "minHeight": true
        }
    }
}
```

---

## Snippet 17 — Disable Full Site Editing Features

```php
// Remove template editing from the editor for classic themes
add_filter( 'gutenberg_can_edit_post_type', '__return_false' );

// Disable the patterns directory (external patterns from wordpress.org)
add_filter( 'should_load_remote_block_patterns', '__return_false' );

// Remove the block directory (install new blocks from editor)
remove_action( 'enqueue_block_editor_assets', 'wp_enqueue_editor_block_directory_assets' );
```

---

## Snippet 18 — Add Custom Post Meta to Block Editor

```php
// Register meta so it's available in the block editor sidebar
add_action( 'init', 'register_post_meta_for_editor' );
function register_post_meta_for_editor(): void {
    register_post_meta( 'post', '_my_subtitle', [
        'show_in_rest'  => true,  // Must be true to appear in editor
        'single'        => true,
        'type'          => 'string',
        'auth_callback' => fn() => current_user_can( 'edit_posts' ),
        'sanitize_callback' => 'sanitize_text_field',
    ]);
}
```

---

## Snippet 19 — Access Post Meta in JavaScript (Editor)

```javascript
// Access registered meta in the block editor
import { useSelect, useDispatch } from '@wordpress/data';

function MyMetaField() {
    const subtitle = useSelect( ( select ) =>
        select( 'core/editor' ).getEditedPostAttribute( 'meta' )._my_subtitle
    );

    const { editPost } = useDispatch( 'core/editor' );

    return (
        <input
            type="text"
            value={ subtitle || '' }
            onChange={ ( e ) =>
                editPost({ meta: { _my_subtitle: e.target.value } })
            }
            placeholder="Enter subtitle..."
        />
    );
}
```

---

## Snippet 20 — Insert Block Programmatically (JavaScript)

```javascript
import { dispatch } from '@wordpress/data';
import { createBlock } from '@wordpress/blocks';

function insertParagraphBlock( content ) {
    const block = createBlock( 'core/paragraph', {
        content: content,
        align: 'center',
    });

    dispatch( 'core/block-editor' ).insertBlocks( block );
}

// Insert after a specific block by clientId
function insertAfterBlock( clientId, content ) {
    const { getBlockIndex } = wp.data.select( 'core/block-editor' );
    const index = getBlockIndex( clientId );

    const block = createBlock( 'core/paragraph', { content } );
    dispatch( 'core/block-editor' ).insertBlocks( block, index + 1 );
}
```

---

## Snippet 21 — Filter Block Output on Frontend

```php
// Modify block HTML output on the frontend
add_filter( 'render_block', 'modify_block_output', 10, 2 );
function modify_block_output( string $block_content, array $block ): string {
    // Add a class to all core/image blocks
    if ( 'core/image' === $block['blockName'] ) {
        $block_content = str_replace(
            'class="wp-block-image',
            'class="wp-block-image my-custom-class',
            $block_content
        );
    }

    return $block_content;
}

// Filter a specific block type only
add_filter( 'render_block_core/paragraph', function( string $content, array $block ): string {
    // Add data attribute to all paragraphs
    return str_replace( '<p', '<p data-block="paragraph"', $content );
}, 10, 2 );
```

---

## Snippet 22 — Add Global Styles via theme.json styles.blocks

```json
{
    "version": 2,
    "styles": {
        "blocks": {
            "core/heading": {
                "typography": {
                    "fontWeight": "700",
                    "lineHeight": "1.2"
                }
            },
            "core/paragraph": {
                "typography": {
                    "fontSize": "var(--wp--preset--font-size--medium)"
                }
            },
            "core/button": {
                "border": {
                    "radius": "4px"
                },
                "spacing": {
                    "padding": {
                        "top":    "12px",
                        "bottom": "12px",
                        "left":   "24px",
                        "right":  "24px"
                    }
                }
            }
        }
    }
}
```

---

## Snippet 23 — Disable Block Editor for Specific Post Types

```php
// Disable Gutenberg for the 'testimonial' post type
add_filter( 'use_block_editor_for_post_type', 'disable_gutenberg_for_post_types', 10, 2 );
function disable_gutenberg_for_post_types( bool $use_editor, string $post_type ): bool {
    $classic_post_types = [ 'testimonial', 'faq', 'team_member' ];

    if ( in_array( $post_type, $classic_post_types, true ) ) {
        return false;
    }

    return $use_editor;
}
```

---

## Snippet 24 — Use InnerBlocks (Allow Child Blocks)

```javascript
// edit.js — create a block that accepts child blocks
import { useBlockProps, InnerBlocks } from '@wordpress/block-editor';

const ALLOWED_BLOCKS = [ 'core/paragraph', 'core/heading', 'core/image' ];
const TEMPLATE = [
    [ 'core/heading',   { level: 2, placeholder: 'Card Title' } ],
    [ 'core/paragraph', { placeholder: 'Card description...' } ],
];

export function Edit() {
    const blockProps = useBlockProps({ className: 'my-card' });

    return (
        <div { ...blockProps }>
            <InnerBlocks
                allowedBlocks={ ALLOWED_BLOCKS }
                template={ TEMPLATE }
                templateLock={ false } // false = user can add/remove blocks
            />
        </div>
    );
}

// save.js
export function Save() {
    const blockProps = useBlockProps.save({ className: 'my-card' });
    return (
        <div { ...blockProps }>
            <InnerBlocks.Content />
        </div>
    );
}
```

---

## Snippet 25 — Unregister Core Blocks You Don't Need

```javascript
// unregister-blocks.js — load this in enqueue_block_editor_assets
import { unregisterBlockType, getBlockTypes } from '@wordpress/blocks';
import { dispatch } from '@wordpress/data';

wp.domReady( () => {
    const blocksToRemove = [
        'core/archives',
        'core/calendar',
        'core/rss',
        'core/search',
        'core/tag-cloud',
        'core/latest-comments',
        'core/verse',
        'core/table-of-contents',
    ];

    blocksToRemove.forEach( ( block ) => {
        try {
            unregisterBlockType( block );
        } catch ( e ) {
            // Block may not exist in this version
        }
    });
});
```
