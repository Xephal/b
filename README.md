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
(async function() {
  const wpData = window.wp && window.wp.data;
  if (!wpData) return console.error('wp.data non trouvé. Ouvre l\'éditeur Gutenberg.');

  const blockSelect = wpData.select('core/block-editor');
  const editorSelect = wpData.select('core/editor');
  const dispatcher = wpData.dispatch('core/block-editor');
  const editorDispatcher = wpData.dispatch('core/editor');

  const postId = wpData.select('core/editor').getCurrentPostId();
  if (!postId) return console.error('Impossible de récupérer l\'ID du post actuel.');

  // Trouve un block core/quote — on prend le premier trouvé
  const blocks = blockSelect.getBlocks();
  const quoteBlock = blocks.find(b => b.name === 'core/quote');

  if (!quoteBlock) {
    console.warn('Aucun block core/quote sur la page. Liste des blocks:', blocks.map(b => b.name).slice(0,10));
    return;
  }

  console.log('Quote block trouvé. clientId:', quoteBlock.clientId, 'attributes avant:', quoteBlock.attributes);

  // Sélectionne le block dans l'éditeur pour visibilité
  dispatcher.selectBlock(quoteBlock.clientId);

  // Tentative d'injection de className (en mémoire puis update)
  const injectedClass = 'pwned-class-' + Math.floor(Math.random()*10000);
  const updatedAttributes = Object.assign({}, quoteBlock.attributes, { className: injectedClass });

  console.log('Injection attempt — mise à jour des attributes en mémoire:', updatedAttributes);
  dispatcher.updateBlock(quoteBlock.clientId, { attributes: updatedAttributes });

  // Force save
  try {
    await editorDispatcher.savePost({ force: true });
    console.log('Save attempted — vérification REST en cours...');
  } catch (e) {
    console.error('Erreur à la sauvegarde:', e);
  }

  // Petite attente pour laisser le serveur répondre
  await new Promise(r => setTimeout(r, 800));

  // Récupère le post via la REST API pour inspecter le HTML rendu
  try {
    const resp = await fetch(`/wp-json/wp/v2/posts/${postId}`, { credentials: 'same-origin' });
    if (!resp.ok) throw new Error('REST fetch failed: ' + resp.status);
    const data = await resp.json();

    console.log('REST response content.rendered (excerpt):', data.content && (data.content.rendered||''));
    // Vérifie la présence de la classe injectée dans le HTML rendu
    const presentInRendered = (data.content && data.content.rendered || '').includes(injectedClass);
    console.log('Classe injectée trouvée dans content.rendered ?', presentInRendered);

    // Inspecte le contenu brut de bloc côté éditeur (si accessible)
    const blocksAfterSave = blockSelect.getBlocks();
    const sameBlock = blocksAfterSave.find(b => b.clientId === quoteBlock.clientId);
    console.log('Attributes côté client après save:', sameBlock ? sameBlock.attributes : 'block non trouvé');

    if (presentInRendered) {
      console.error('ATTENTION — la classe a été persistée côté serveur !');
    } else {
      console.log('OK — la classe n\'est pas visible dans le HTML rendu. Mu-plugin semble avoir neutralisé l\'injection.');
    }

  } catch (err) {
    console.error('Erreur lors de la vérification REST:', err);
  }

  // Nettoyage visuel : retirer l'attribut dans l'éditeur si il reste en mémoire
  try {
    // restore original attributes (remove className)
    const cleaned = Object.assign({}, quoteBlock.attributes);
    delete cleaned.className;
    dispatcher.updateBlock(quoteBlock.clientId, { attributes: cleaned });
    console.log('Tentative de restauration locale effectuée.');
  } catch(e) { /* silent */ }

})();

```
