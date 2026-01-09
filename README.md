Très bien. Là on fait du béton, pas du PowerPoint. Deux fichiers, immuables, ennuyeux, fiables. Exactement ce qu’on veut à ce stade.

Je te donne le code prêt à commit, sans fioritures, sans “on verra plus tard”.

⸻

📄 src/Application/Common/Period/Period.php

Rôle
	•	Value Object de période
	•	from / to inclusifs
	•	Aucune logique métier
	•	Aucune dépendance HTTP
	•	Immuable

<?php

declare(strict_types=1);

namespace App\Application\Common\Period;

use DateTimeImmutable;
use InvalidArgumentException;

final class Period
{
    private DateTimeImmutable $from;
    private DateTimeImmutable $to;

    public function __construct(DateTimeImmutable $from, DateTimeImmutable $to)
    {
        if ($from > $to) {
            throw new InvalidArgumentException('Period "from" date must be before or equal to "to" date.');
        }

        $this->from = $from;
        $this->to = $to;
    }

    public function from(): DateTimeImmutable
    {
        return $this->from;
    }

    public function to(): DateTimeImmutable
    {
        return $this->to;
    }

    /**
     * Durée en jours calendaires, bornes incluses.
     * Ex: 01 → 09 = 9 jours
     */
    public function lengthInDays(): int
    {
        return (int) $this->from
            ->diff($this->to)
            ->days + 1;
    }
}

Pourquoi c’est volontairement simple
	•	Pas de timezone ici
	•	Pas de “now”
	•	Pas de calcul de période précédente
	•	Pas de weekday

👉 Ce fichier ne changera presque jamais. C’est exactement ce qu’on veut.

⸻

📄 src/Application/Common/Period/ResolvedPeriod.php

Rôle
	•	Conteneur immuable
	•	Porte toujours deux périodes :
	•	courante
	•	comparaison
	•	Aucun calcul
	•	Aucune règle métier

<?php

declare(strict_types=1);

namespace App\Application\Common\Period;

final class ResolvedPeriod
{
    private Period $current;
    private Period $comparison;

    public function __construct(Period $current, Period $comparison)
    {
        $this->current = $current;
        $this->comparison = $comparison;
    }

    public function current(): Period
    {
        return $this->current;
    }

    public function comparison(): Period
    {
        return $this->comparison;
    }
}


⸻

🧠 Pourquoi ces fichiers sont “terminés” dès maintenant
	•	✔ aucune dépendance instable
	•	✔ pas couplés à un use case
	•	✔ pas couplés à Symfony
	•	✔ testables trivialement
	•	✔ lisibles dans 2 ans

Ils servent juste à porter un contrat. Et ils le font bien.

⸻

⛔ Ce qu’on n’a volontairement PAS mis
	•	DatePeriod PHP (trop implicite)
	•	méthodes utilitaires douteuses
	•	logique de comparaison
	•	règles par défaut
	•	weekdays

Tout ça appartient au PeriodResolver, pas ici.

⸻

✅ Prochaine étape logique

👉 PeriodResolver.php

Là on va :
	•	lire la Request
	•	appliquer la règle métier par défaut
	•	calculer la période précédente exactement comme ta frise

Dis-le et on l’écrit ligne par ligne, proprement, sans magie.
