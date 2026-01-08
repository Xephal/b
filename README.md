Tu as raison: **dans TON code actuel**, le filtrage le plus simple ne se fait pas sur un JSON “date[] + séries[]” (comme je te proposais avant), mais **sur ta timeline `$return`** que tu construis déjà dans `getUsageData()` (clé = `d/m/Y`). Donc on va jouer avec ça. Minimal. Sans refaire ton monde.

## Patch minimal à ajouter (1 seul endroit utile)

### 1) Fais passer `weekday` jusqu’à `getUsageData`

Dans `dataForChart()` (tu l’as déjà dans ton message texte, mais sur ta photo tu ne le passes pas), mets juste ça :

```php
#[Route('/chart_data', name: 'admin_chart_data')]
public function dataForChart(Request $request): Response
{
    $fromDate = $this->getFromDate($request, 'from_date');
    $toDate   = $this->getToDate($request, 'to_date');

    $weekday = $request->query->get('weekday');
    $weekday = $weekday !== null ? (int) $weekday : null;

    $export = $this->getUsageData($fromDate, $toDate, true, $weekday);

    return $this->json($export);
}
```

### 2) Ajoute le paramètre dans la signature de `getUsageData`

```php
protected function getUsageData(
    ?\DateTimeInterface $fromDate,
    ?\DateTimeInterface $toDate,
    bool $forChart = false,
    ?int $weekday = null
): array {
    // ...
}
```

### 3) Filtre **pendant** la construction de `$return` (le vrai fix)

Dans ton `while ($current <= $toDate) { ... }`, tu ajoutes **exactement** ce `if` au tout début de la boucle :

```php
while ($current <= $toDate) {

    if ($weekday !== null && (int) $current->format('N') !== $weekday) {
        $current = (clone $current)->modify('+1 day');
        continue;
    }

    $day = $current->format('d/m/Y');

    $return[$day] = [
        'messagesPerDay' => 0,
        'activeUsersPerDay' => 0,
        'avgResponseTimePerDay' => 0,
        'connectionPerDay' => 0,
        'conversationPerDay' => 0,
    ];

    $current = (clone $current)->modify('+1 day');
}
```

**C’est tout.**

## Pourquoi ça marche avec ton code (et pas “rien à voir”)

* Tu construis déjà la timeline `$return` avec **tous les jours**.
* `prepareData()` ne remplit que si `isset($return[$type['day']])`.
* Donc si tu n’as dans `$return` que les **jeudis**, tu ne rempliras que ces jours-là, et le reste n’existera même pas.

Résultat: tes charts n’auront que `1 et 8 janvier` (si ce sont des jeudis), exactement ce que tu veux.

## Convention weekday

Le `format('N')` de PHP retourne :

* 1 = lundi … 7 = dimanche
  Donc ton dropdown doit envoyer `1..7`.

Voilà le code minimal. Pas de refacto, pas de helper exotique, pas de “source unique”, juste un `if` bien placé.
