Maintenant, on arrête d’empiler et on fait circuler le courant.
Tu as le moteur. Il faut brancher la prise. Étape suivante unique et logique :

👉 exposer le use case via le controller admin.
Pas de front, pas de Twig cosmétique, juste un endpoint propre qui prouve que tout ce qu’on a fait tient debout.

⸻

🧭 Ce qu’on fait maintenant (et seulement ça)

Objectif
	•	Valider tout le pipeline
	•	Depuis la Request
	•	Jusqu’au DTO final
	•	Sans EasyAdmin, sans React, sans décor

⸻

📄 Étape 1 — Modifier DashboardController

📁 src/Controller/Admin/DashboardController.php

On ajoute une action dédiée aux métriques users.

⸻

✅ Action controller exacte (copier-coller)

<?php

declare(strict_types=1);

namespace App\Controller\Admin;

use App\Application\Admin\UseCase\GetUserMetrics;
use App\Application\Admin\UseCase\GetUserMetricsHandler;
use App\Application\Common\Period\PeriodResolver;
use DateTimeImmutable;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Annotation\Route;

final class DashboardController extends AbstractController
{
    #[Route('/admin/kpi/users', name: 'admin_kpi_users', methods: ['GET'])]
    public function userMetrics(
        Request $request,
        PeriodResolver $periodResolver,
        GetUserMetricsHandler $handler,
    ): JsonResponse {
        // 1️⃣ Résoudre les périodes (courante + comparaison)
        $resolvedPeriod = $periodResolver->resolve(
            $request,
            new DateTimeImmutable()
        );

        // 2️⃣ Lire le filtre weekday (optionnel)
        $weekdays = $request->query->all('weekdays');
        $weekdays = $weekdays !== [] ? array_map('intval', $weekdays) : null;

        // 3️⃣ Exécuter le use case
        $result = $handler->handle(
            new GetUserMetrics(
                $resolvedPeriod,
                $weekdays
            )
        );

        // 4️⃣ Retour brut (on veut voir la vérité)
        return $this->json($result);
    }
}


⸻

🧠 Pourquoi cette action est “propre”

✔ Le controller orchestre, point
	•	lit la Request
	•	appelle le resolver
	•	appelle le handler
	•	retourne le résultat

Aucune logique métier.
Aucune logique SQL.
Aucune logique UI.

⸻

✔ Tu peux tester immédiatement

Appelle dans ton navigateur ou Postman :

/admin/kpi/users

ou avec filtres :

/admin/kpi/users?date_from=2025-01-10&date_to=2025-01-18&weekdays[]=1&weekdays[]=2

👉 Tu dois obtenir :
	•	les counts
	•	les évolutions %
	•	les périodes courante / comparaison

⸻

🚨 Si quelque chose ne marche pas maintenant

C’est forcément :
	•	un mapping Doctrine
	•	un champ createdAt
	•	une relation mal nommée

👉 Et c’est parfait, parce que :
	•	tout est localisé
	•	rien n’est caché
	•	pas de front pour brouiller les pistes

⸻

🧪 Étape de validation (à faire avant de continuer)
	•	endpoint répond
	•	JSON cohérent
	•	périodes correctes
	•	évolution logique
	•	aucun warning PHP

⸻

🧭 Et après seulement

Quand ce endpoint est validé :

Étape suivante possible

👉 Afficher la période de comparaison sous le date picker (Twig)

Puis seulement après :
	•	composant React
	•	binding avec Stimulus
	•	polish UI

⸻

Verdict honnête

Tu es exactement au point où beaucoup arrêtent trop tôt.
Toi, tu fais circuler le flux jusqu’au bout. C’est bien.

Dis-moi :
	•	ce que retourne l’endpoint
	•	ou l’erreur exacte si ça casse

Et on corrige chirurgicalement.
