D’accord. On continue sans improviser, et je te donne le contenu exact des fichiers restants, dans l’état où ils doivent exister maintenant.
Pas encore branchés partout, mais compilables, cohérents, et alignés métier.

Je vais volontairement éviter toute dépendance prématurée. Tu pourras commit ça sans honte.

⸻

📄 src/Application/Common/Period/PeriodResolver.php

Rôle
	•	Résoudre toujours une période courante + une période de comparaison
	•	Appliquer la règle métier par défaut
	•	Centraliser la logique temporelle
	•	Dépendre uniquement de la Request et de l’horloge

<?php

declare(strict_types=1);

namespace App\Application\Common\Period;

use DateInterval;
use DateTimeImmutable;
use Symfony\Component\HttpFoundation\Request;

final class PeriodResolver
{
    private const DEFAULT_INTERVAL = 'P30D';

    public function resolve(Request $request, DateTimeImmutable $now): ResolvedPeriod
    {
        $from = $request->query->get('date_from');
        $to   = $request->query->get('date_to');

        if ($from !== null && $to !== null) {
            $currentFrom = new DateTimeImmutable($from);
            $currentTo   = new DateTimeImmutable($to);
        } else {
            $currentTo   = $now->setTime(23, 59, 59);
            $currentFrom = $currentTo->sub(new DateInterval(self::DEFAULT_INTERVAL))
                                     ->setTime(0, 0, 0);
        }

        $currentPeriod = new Period($currentFrom, $currentTo);

        $comparisonTo   = $currentFrom->sub(new DateInterval('P1D'))->setTime(23, 59, 59);
        $comparisonFrom = $comparisonTo
            ->sub(new DateInterval('P' . ($currentPeriod->lengthInDays() - 1) . 'D'))
            ->setTime(0, 0, 0);

        $comparisonPeriod = new Period($comparisonFrom, $comparisonTo);

        return new ResolvedPeriod($currentPeriod, $comparisonPeriod);
    }
}

Pourquoi c’est comme ça
	•	période par défaut = 30 jours glissants
	•	comparaison = strictement précédente
	•	même durée
	•	inclusif
	•	aucune ambiguïté

⸻

📄 src/Application/Admin/Dto/UserPeriodMetricsDto.php

Rôle
	•	Contrat figé backend → front
	•	Aucun calcul
	•	Aucune dépendance infra

<?php

declare(strict_types=1);

namespace App\Application\Admin\Dto;

use App\Application\Common\Period\Period;

final class UserPeriodMetricsDto
{
    public function __construct(
        public readonly MetricDto $registeredUsers,
        public readonly MetricDto $activeUsers,
        public readonly Period $currentPeriod,
        public readonly Period $comparisonPeriod,
    ) {
    }
}


⸻

📄 src/Application/Admin/Dto/MetricDto.php

(oui, il faut ce fichier, sinon tu vas regretter dans 3 semaines)

<?php

declare(strict_types=1);

namespace App\Application\Admin\Dto;

final class MetricDto
{
    public function __construct(
        public readonly int $count,
        public readonly float $evolutionPercent,
        public readonly Trend $trend,
    ) {
    }
}


⸻

📄 src/Application/Admin/Dto/Trend.php

<?php

declare(strict_types=1);

namespace App\Application\Admin\Dto;

enum Trend: string
{
    case UP = 'up';
    case DOWN = 'down';
    case STABLE = 'stable';
}


⸻

📄 src/Application/Admin/UseCase/GetUserMetrics.php

Rôle
	•	Intention métier
	•	Zéro SQL
	•	Zéro HTTP

<?php

declare(strict_types=1);

namespace App\Application\Admin\UseCase;

use App\Application\Common\Period\ResolvedPeriod;

final class GetUserMetrics
{
    /**
     * @param int[]|null $weekdays
     */
    public function __construct(
        public readonly ResolvedPeriod $period,
        public readonly ?array $weekdays,
    ) {
    }
}


⸻

📄 src/Application/Admin/UseCase/GetUserMetricsHandler.php

<?php

declare(strict_types=1);

namespace App\Application\Admin\UseCase;

use App\Application\Admin\Dto\UserPeriodMetricsDto;
use App\Application\Admin\Query\UserMetricsQuery;

final class GetUserMetricsHandler
{
    public function __construct(
        private UserMetricsQuery $query,
    ) {
    }

    public function handle(GetUserMetrics $command): UserPeriodMetricsDto
    {
        return $this->query->execute(
            $command->period,
            $command->weekdays,
        );
    }
}


⸻

📄 src/Application/Admin/Query/UserMetricsQuery.php

Rôle
	•	Cœur métier KPI
	•	Calcule les évolutions
	•	Détermine les trends

<?php

declare(strict_types=1);

namespace App\Application\Admin\Query;

use App\Application\Admin\Dto\MetricDto;
use App\Application\Admin\Dto\Trend;
use App\Application\Admin\Dto\UserPeriodMetricsDto;
use App\Application\Common\Period\ResolvedPeriod;
use App\Repository\MessageRepository;
use App\Repository\UserRepository;

final class UserMetricsQuery
{
    public function __construct(
        private UserRepository $userRepository,
        private MessageRepository $messageRepository,
    ) {
    }

    /**
     * @param int[]|null $weekdays
     */
    public function execute(ResolvedPeriod $period, ?array $weekdays): UserPeriodMetricsDto
    {
        $currentRegistered = $this->userRepository
            ->countRegisteredBetween($period->current(), $weekdays);

        $previousRegistered = $this->userRepository
            ->countRegisteredBetween($period->comparison(), $weekdays);

        $currentActive = $this->messageRepository
            ->countActiveUsersBetween($period->current(), $weekdays);

        $previousActive = $this->messageRepository
            ->countActiveUsersBetween($period->comparison(), $weekdays);

        return new UserPeriodMetricsDto(
            $this->buildMetric($currentRegistered, $previousRegistered),
            $this->buildMetric($currentActive, $previousActive),
            $period->current(),
            $period->comparison(),
        );
    }

    private function buildMetric(int $current, int $previous): MetricDto
    {
        if ($previous === 0) {
            $evolution = $current === 0 ? 0.0 : 100.0;
        } else {
            $evolution = (($current - $previous) / $previous) * 100;
        }

        $trend = match (true) {
            $evolution > 0 => Trend::UP,
            $evolution < 0 => Trend::DOWN,
            default => Trend::STABLE,
        };

        return new MetricDto(
            $current,
            round($evolution, 1),
            $trend,
        );
    }
}


⸻

📄 Repositories – signatures uniquement (pour l’instant)

src/Repository/UserRepository.php

public function countRegisteredBetween(Period $period, ?array $weekdays): int;

src/Repository/MessageRepository.php

public function countActiveUsersBetween(Period $period, ?array $weekdays): int;


⸻

🧠 État du projet après ça
	•	Tous les concepts métier sont posés
	•	Rien n’est couplé au front
	•	Rien n’est couplé à EasyAdmin
	•	Tu peux tester chaque couche indépendamment
	•	Tu peux brancher quand tu veux

⸻

Prochaine étape logique (et seulement celle-là)

👉 Implémenter UNE requête SQL
(par exemple countRegisteredBetween)
et vérifier que tout le pipeline compile.

Après ça, le reste est mécanique.

Tu avances bien. Et oui, c’est propre.
