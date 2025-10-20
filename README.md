🎯 Objectif

Empêcher les appels à scrollIntoView() présents dans MessageElement.tsx de provoquer un autoscroll non désiré pendant la génération (loading = true) ou quand l’utilisateur a scrolled up.

📂 Fichier à modifier

src/components/chat/MessageElement.tsx

🔧 Étapes détaillées
1️⃣ Identifier tous les appels à scrollIntoView

Rechercher dans le fichier les occurrences de :

.scrollIntoView(


Elles apparaissent généralement dans :

useEffect (ex: quand isLiked ou isDisliked changent)

callbacks axios (handleRequestLikeDislike, handleFormSubmit, etc.)

éventuellement dans d’autres handlers (feedback, export, etc.)

2️⃣ Ajouter un utilitaire commun en haut du fichier

Juste avant le premier useEffect, insérer :

// Utilitaire global pour savoir si on est proche du bas du conteneur scrollable
const findScrollContainer = (el: HTMLElement | null): HTMLElement | null => {
  let current: HTMLElement | null = el?.parentElement || null
  while (current) {
    const style = getComputedStyle(current)
    const overflowY = style.overflowY
    if (overflowY === 'auto' || overflowY === 'scroll') return current
    current = current.parentElement
  }
  return null
}

const isNearBottom = (ref: React.RefObject<HTMLElement>, threshold = 50): boolean => {
  const container = findScrollContainer(ref.current)
  if (!container) return true
  return container.scrollTop + container.clientHeight >= container.scrollHeight - threshold
}

3️⃣ Envelopper chaque scrollIntoView existant

Pour chaque ligne du type :

bottomOfFeedbackRef.current?.scrollIntoView({ behavior: 'smooth' })


Remplacer par :

if (isNearBottom(bottomOfFeedbackRef)) {
  bottomOfFeedbackRef.current?.scrollIntoView({ behavior: 'smooth' })
}


Répéter pour tous les refs :

bottomOfSourceRef

bottomOfFeedbackRef

tout autre ref utilisé avec scrollIntoView()

4️⃣ (Optionnel) Si loading est dans le scope, renforcer la condition :

Dans les handlers liés à des événements pendant un stream, ajouter :

if (!loading && isNearBottom(bottomOfFeedbackRef)) {
  bottomOfFeedbackRef.current?.scrollIntoView({ behavior: 'smooth' })
}

5️⃣ Vérification automatique

À la fin, ton agent peut exécuter un check rapide :

grep -R "scrollIntoView" src/components/chat/MessageElement.tsx


et vérifier que tous les appels sont précédés d’un if (isNearBottom(...)).

✅ Critère de réussite

Pendant la génération (loading = true), l’autoscroll reste stable sauf si l’utilisateur scrolle manuellement.

Si l’utilisateur scrolle vers le haut, aucun like/dislike/comment ne ramène la vue en bas.

Quand un nouveau message commence (loading repasse à true), l’autoscroll se réactive normalement.
