```
<?php
// wp-content/mu-plugins/disable-custom-class.php
// Minimal MU plugin => disable additional CSS classes in Gutenberg (server + editor fallback)

/**
 * Server-side: forcefully disable customClassName support for all blocks at registration time.
 * This ensures no block will save a custom className even if some client hack tries to re-enable it.
 */
add_filter('register_block_type_args', function(array $args, $name) : array {
    if (! isset($args['supports']) || ! is_array($args['supports'])) {
        $args['supports'] = [];
    }
    $args['supports']['customClassName'] = false;
    return $args;
}, 10, 2);

/**
 * Editor assets: inject JS that removes customClassName support client-side so the UI vanishes,
 * and inject some defensive CSS/DOM removal as a fallback.
 */
add_action('enqueue_block_editor_assets', function() {
    // Inline JS: use wp.hooks to disable customClassName on client for all blocks,
    // plus a DOM-cleanup fallback that removes the Advanced control if still present.
    $js = <<<'JS'
( function (wp) {
    if (!wp || !wp.hooks || !wp.data) return;

    // 1) Hook into block registration -> remove client-side support so UI won't be rendered
    wp.hooks.addFilter(
        'blocks.registerBlockType',
        'disable-custom-class/namespace',
        function (settings, name) {
            settings.supports = settings.supports || {};
            settings.supports.customClassName = false;
            return settings;
        }
    );

    // 2) Fallback DOM cleanup: remove existing Advanced "Additional CSS class(es)" controls
    //    Works by scanning labels that match typical texts in EN/FR (robuste-ish).
    const removeAdvancedField = function() {
        try {
            const labels = document.querySelectorAll('.components-base-control__label, .components-base-control label');
            labels.forEach(label => {
                const txt = (label.textContent || '').trim().toLowerCase();
                if (!txt) return;
                // English / French hints
                if (txt.includes('additional css') || txt.includes('additional css class') ||
                    txt.includes('classe(s) css') || txt.includes('classe css') || txt.includes('css addition')) {
                    const control = label.closest('.components-base-control') || label.closest('.components-panel__row') || label.parentElement;
                    if (control && control.remove) control.remove();
                }
            });
        } catch (e) {
            // silent: don't break the editor
            // console.error(e);
        }
    };

    // Run once and also after mutations (editor renders dynamically)
    removeAdvancedField();
    const observer = new MutationObserver(function() {
        removeAdvancedField();
    });
    const root = document.querySelector('.edit-post-sidebar, .edit-post-layout__content, document.body');
    observer.observe(document.body, { childList: true, subtree: true });
})(window.wp);
JS;

    // Register a small handle and inject inline script. Using an empty src for MU plugin inline is fine.
    // Some WP installs may refuse empty src; register with false then add inline.
    wp_register_script('mu-disable-custom-class', '', ['wp-hooks', 'wp-data'], false, true);
    wp_enqueue_script('mu-disable-custom-class');
    wp_add_inline_script('mu-disable-custom-class', $js);

    // Defensive CSS to hide leftover UI (selectors are intentionally generic and !important)
    $css = <<<'CSS'
/* Defensive hide of the "Additional CSS class(es)" control in Inspector (en/fr) */
.editor-styles-wrapper .components-base-control__help,
.editor-styles-wrapper .components-base-control__field,
.components-panel__body .components-base-control {
    /* do not accidentally hide everything; rely on JS too - keep rule narrow */
}
.components-base-control__label {
    /* hide labels that match the pattern via JS fallback; CSS can't match text reliably */
}
/* If you want to aggressively hide known classes, add them here, but prefer the JS/server approach above. */
CSS;
    // Use existing editor style handle to inject CSS inline
    wp_add_inline_style('wp-edit-blocks', $css);
});
```

```
// Dans la console (Editor open)
const { select, dispatch } = wp.data;
const blocks = select('core/block-editor').getBlocks();
if (!blocks.length) console.log('pas de blocks sur la page');
else {
  const first = blocks[0];
  // force injection de className en mémoire et sauvegarde
  const edited = { ...first, attributes: { ...first.attributes, className: 'pwned-class' } };
  dispatch('core/block-editor').updateBlock(edited.clientId, edited);
  // sauvegarde du post
  dispatch('core/editor').savePost({ force: true }).then(()=>console.log('save attempted'));
}
```
