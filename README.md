Ah, je vois. Tu veux juste que les explications s’affichent à nouveau sans démolir ta politique CSP. Pas besoin de tout recompiler l’écosystème du plugin — on va juste lui rendre la capacité d’afficher la question explanation, sans lever la barrière de sécurité.

Le problème est simple :
👉 le plugin utilise du JavaScript inline pour faire apparaître les explications (souvent avec un onclick="ays_show_explanation()" ou via un script eval() en fin de HTML).
Ta CSP bloque tout ça, donc la fonction ne s’exécute pas. Résultat : l’élément .ays_questtion_explanation reste en display:none.

On va juste faire ce qu’il aurait dû faire proprement :
activer ces explications avec un JS statique sous ta CSP.

⸻

🧱 Étape — objectif clair

Réactiver l’affichage de .ays_questtion_explanation (les explications post-réponse) sans modifier la CSP ni autoriser de script inline.

⸻

🧪 Test rouge — avant correctif

Dans le front, réponds à une question :
tu vois que la zone d’explication existe dans le DOM (.ays_questtion_explanation) mais reste masquée.
Inspecte-la : display: none; → c’est notre test rouge.

⸻

🩹 Patch minimal — un petit JS statique sûr CSP

Crée ce fichier :

/wp-content/uploads/quizmaker-csp/quizmaker-fix-explanation.js

Avec ce contenu :

document.addEventListener("DOMContentLoaded", () => {
  // Sélectionne les zones d'explication masquées
  const explanations = document.querySelectorAll(".ays_questtion_explanation");

  if (!explanations.length) return;

  // Quand un quiz est répondu, le plugin ajoute souvent une classe ou un attribut 'data-answered' ou 'data-status'
  // On observe les changements pour les révéler proprement
  const observer = new MutationObserver(() => {
    explanations.forEach(exp => {
      // Si l'élément est dans le DOM et qu'il a un texte (non vide)
      if (exp.textContent.trim().length > 0) {
        exp.style.display = "block";
        exp.style.visibility = "visible";
      }
    });
  });

  observer.observe(document.body, { childList: true, subtree: true });

  // fallback : au clic sur un bouton "Next"/"Finish", on vérifie aussi
  document.body.addEventListener("click", e => {
    if (e.target.matches(".ays_next, .ays_finish, button")) {
      setTimeout(() => {
        document.querySelectorAll(".ays_questtion_explanation").forEach(exp => {
          if (exp.textContent.trim().length > 0) {
            exp.style.display = "block";
            exp.style.visibility = "visible";
          }
        });
      }, 300);
    }
  });
});

Ensuite, dans ton MU-plugin (celui que tu utilises déjà pour tes corrections CSP), ajoute à la fin :

add_action('wp_enqueue_scripts', function() {
    wp_enqueue_script(
        'quizmaker-fix-explanation',
        content_url('uploads/quizmaker-csp/quizmaker-fix-explanation.js'),
        [],
        '1.0.0',
        true
    );
}, 30);


⸻

⚙️ Pourquoi
	•	Ce script ne dépend d’aucun inline JS, donc aucune violation CSP.
	•	Il se contente de détecter la présence de .ays_questtion_explanation et de forcer son affichage quand elle contient du texte.
	•	Il marche même si le plugin change un peu son DOM : on observe les mutations.

⸻

🚫 Anti-patterns évités
	•	Pas de unsafe-inline ou nonce bidon dans ta CSP.
	•	Pas de modification du core du plugin.
	•	Pas d’écrasement des comportements existants : on ne fait qu’afficher ce qui aurait dû être visible après réponse.

⸻

✅ Étape suivante

Recharge la page du quiz avec la CSP active, réponds à une question → l’explication devrait apparaître.
Si ça ne bouge toujours pas, donne-moi un extrait exact du HTML de ton .ays_questtion_explanation (avec le parent qui l’entoure), et je t’ajusterai le sélecteur ou le déclencheur sans te demander de relâcher la sécurité.