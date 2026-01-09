aTrès bien. Là on fait le reset cérébral que Doctrine t’a forcé à faire.
Voici TOUS les bouts de code à mettre à jour, fichier par fichier, sans trous, pour que :
	•	plus aucun IN ()
	•	plus aucun DAYOFWEEK inconnu
	•	plus aucun assert($lookahead !== null)
	•	plus aucune ambiguïté de type

Tu appliques exactement ça, dans cet ordre, et on n’en parle plus.

⸻

1️⃣ DashboardController.php

📁 src/Controller/Admin/DashboardController.php

🔁 À REMPLACER — normalisation de weekdays

❌ Ancien code (fragile)

$weekdays = $request->query->all('weekdays');
$weekdays = $weekdays !== [] ? array_map('intval', $weekdays) : null;

✅ Nouveau code (robuste, définitif)

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

🎯 Résultat garanti :
	•	$weekdays === null
	•	OU $weekdays = int[] non vide

⸻

2️⃣ UserRepository.php

📁 src/Repository/UserRepository.php

🔁 À METTRE À JOUR — clause weekday DQL

❌ Ancien code (cassé)

->andWhere('DAYOFWEEK(u.createdAt) IN (:weekdays)')

✅ Nouveau code (DQL valide)

->andWhere("FUNCTION('DAYOFWEEK', u.createdAt) IN (:weekdays)")


⸻

🔁 À METTRE À JOUR — condition de garde

❌ Ancien code (pas assez strict)

if (!empty($weekdays)) {

✅ Nouveau code (anti IN ())

if ($weekdays !== null && count($weekdays) > 0) {


⸻

🔁 Code final à avoir dans UserRepository

if ($weekdays !== null && count($weekdays) > 0) {
    $qb
        ->andWhere("FUNCTION('DAYOFWEEK', u.createdAt) IN (:weekdays)")
        ->setParameter('weekdays', $this->mapWeekdaysToSql($weekdays));
}


⸻

3️⃣ MessageRepository.php

📁 src/Repository/MessageRepository.php

👉 Exactement les mêmes corrections, mais sur m.createdAt

⸻

🔁 DQL weekday

->andWhere("FUNCTION('DAYOFWEEK', m.createdAt) IN (:weekdays)")


⸻

🔁 Condition de garde

if ($weekdays !== null && count($weekdays) > 0) {


⸻

🔁 Code final à avoir dans MessageRepository

if ($weekdays !== null && count($weekdays) > 0) {
    $qb
        ->andWhere("FUNCTION('DAYOFWEEK', m.createdAt) IN (:weekdays)")
        ->setParameter('weekdays', $this->mapWeekdaysToSql($weekdays));
}


⸻

4️⃣ (OPTIONNEL MAIS RECOMMANDÉ)

Sécuriser getSingleScalarResult()

Si tu veux être blindé contre les jeux de données vides :

🔁 Remplacer

return (int) $qb->getQuery()->getSingleScalarResult();

✅ Par

$result = $qb->getQuery()->getOneOrNullResult();

return $result !== null ? (int) array_values($result)[0] : 0;

Pas obligatoire si les guards sont bien en place, mais ça rend Doctrine moins susceptible.

⸻

5️⃣ CHECKLIST FINALE (à cocher)
	•	DashboardController normalise weekdays
	•	weekdays est toujours null ou int[] non vide
	•	FUNCTION('DAYOFWEEK', …) utilisé partout
	•	AUCUN IN () possible
	•	Même logique dans User + Message repositories
	•	/admin/kpi/users répond sans erreur

⸻

🧠 Verdict

Tu viens de traverser le triangle des Bermudes Doctrine :
	•	types HTTP
	•	DQL vs SQL
	•	paramètres conditionnels

Si ça marche maintenant, ça marchera toujours.

👉 Dis-moi juste :
	•	“OK ça marche”
	•	ou colle le JSON retourné

Après ça, on passe au Twig / affichage période / React, qui va te sembler ridiculement simple après ce cirque.
