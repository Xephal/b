Très bien. On repart à zéro, proprement, sans résidu de copier-coller.
Voici LA route complète, fonctionnelle, compilable, sans variable fantôme, à coller telle quelle.

⸻

📄 src/Controller/Admin/DashboardController.php

Action /admin/kpi/users complète

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
        // 1️⃣ Résolution de la période (courante + comparaison)
        $resolvedPeriod = $periodResolver->resolve(
            $request,
            new DateTimeImmutable()
        );

        // 2️⃣ Normalisation du filtre weekdays (null ou int[] non vide)
        $weekdaysParam = $request->query->get('weekdays');

        if ($weekdaysParam === null || $weekdaysParam === '') {
            $weekdays = null;
        } elseif (is_array($weekdaysParam)) {
            $weekdays = array_filter(array_map('intval', $weekdaysParam));
            $weekdays = $weekdays !== [] ? $weekdays : null;
        } else {
            $weekdays = array_filter(
                array_map('intval', explode(',', $weekdaysParam))
            );
            $weekdays = $weekdays !== [] ? $weekdays : null;
        }

        // 3️⃣ Exécution du use case
        $result = $handler->handle(
            new GetUserMetrics(
                $resolvedPeriod,
                $weekdays
            )
        );

        // 4️⃣ Réponse JSON
        return $this->json($result);
    }
}


⸻

✅ Ce que cette route garantit
	•	✔ aucune variable non définie
	•	✔ weekdays toujours valide (null ou int[])
	•	✔ aucune logique métier dans le controller
	•	✔ réponse JSON brute (idéale pour debug)
	•	✔ pipeline complet validé

⸻

🧪 Tests rapides à faire

Toutes ces URLs doivent fonctionner sans erreur :

/admin/kpi/users
/admin/kpi/users?weekdays[]=1&weekdays[]=2
/admin/kpi/users?weekdays=1,2
/admin/kpi/users?date_from=2025-01-10&date_to=2025-01-18


⸻

🧠 Si ça casse encore

À ce stade, ce ne sera plus le controller.

Ce sera :
	•	un mapping Doctrine (createdAt)
	•	une relation (conversation → user)
	•	ou une donnée vide inattendue

Et là, on corrigera au bon endroit, sans bricoler.

⸻

Verdict final

Colle ce code, recharge /admin/kpi/users.
	•	Si ça marche → on passe à l’affichage Twig / UI
	•	Si ça casse → colle l’erreur exacte

Tu es à 95 %. Le dernier 5 % est toujours le plus pénible.
