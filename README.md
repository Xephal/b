Oui, on avance bien. Et non, ce n’est pas bizarre, c’est logique. Tu viens juste de tomber sur le dernier détail invisible mais crucial : la sérialisation JSON des objets Period.

Respire. Les chiffres sont bons. C’est l’emballage qui est vide.

⸻

🔍 Ce que tu observes (et pourquoi)

Dans ton JSON, tu as :

"currentPeriod": [],
"comparisonPeriod": []

👉 Ça ne veut pas dire que les périodes sont vides.
👉 Ça veut dire que Symfony ne sait pas comment sérialiser tes objets Period.

Pourquoi ?
	•	Period est un value object
	•	avec des propriétés privées
	•	sans getters exposés au serializer
	•	sans implémentation de JsonSerializable

Résultat :

Symfony dit “ok, je vois un objet… je n’ai rien à en sortir” → {} / []

⸻

🧠 Règle importante (à retenir)

Un DTO exposé en JSON ne doit jamais contenir directement un Value Object non sérialisable.

Tu as fait architecturalement juste, maintenant on fait techniquement lisible.

⸻

✅ La solution propre (et la bonne)

On ne touche pas :
	•	aux repositories
	•	aux use cases
	•	aux calculs
	•	aux règles métier

On fait exactement UNE chose :
👉 rendre Period sérialisable explicitement.

⸻

🛠️ Option recommandée : implémenter JsonSerializable

📄 src/Application/Common/Period/Period.php

Ajoute l’interface et la méthode suivante.

✅ Version corrigée complète

<?php

declare(strict_types=1);

namespace App\Application\Common\Period;

use DateTimeImmutable;
use InvalidArgumentException;
use JsonSerializable;

final class Period implements JsonSerializable
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

    public function lengthInDays(): int
    {
        return (int) $this->from
            ->diff($this->to)
            ->days + 1;
    }

    public function jsonSerialize(): array
    {
        return [
            'from' => $this->from->format('Y-m-d'),
            'to'   => $this->to->format('Y-m-d'),
        ];
    }
}

👉 C’est tout.

⸻

🧪 Ce que tu dois voir après ça

Recharge exactement la même URL :

/admin/kpi/users?date_from=2025-12-10&date_to=2025-12-18&weekdays[]=1&weekdays[]=2

Tu dois maintenant obtenir quelque chose comme :

{
  "registeredUsers": {
    "count": 708,
    "evolutionPercent": 100,
    "trend": "up"
  },
  "activeUsers": {
    "count": 514,
    "evolutionPercent": 100,
    "trend": "up"
  },
  "currentPeriod": {
    "from": "2025-12-10",
    "to": "2025-12-18"
  },
  "comparisonPeriod": {
    "from": "2025-12-01",
    "to": "2025-12-09"
  }
}

👉 Et là, soudainement, tout devient interprétable :
	•	tu vois la frise
	•	tu vois la comparaison
	•	le +100 % a un contexte
	•	l’UI peut afficher la période sous le date picker

⸻

🚫 Ce qu’on ne fait surtout pas
	•	❌ calculer les périodes dans le front
	•	❌ reconstruire les dates côté JS
	•	❌ exposer des DateTimeImmutable bruts
	•	❌ “deviner” la période de comparaison

Tout est déjà juste côté backend.

⸻

Verdict
	•	✔ les calculs sont bons
	•	✔ le pipeline est sain
	•	✔ il manquait uniquement la sérialisation explicite
	•	✔ problème classique, bien attrapé

👉 Fais cette modif, recharge, et dis-moi si tu vois bien les périodes.
Après ça, on passe à l’affichage Twig sous le date picker, et là tu vas enfin “voir” ton boulot.
