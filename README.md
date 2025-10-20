```
import React, { useEffect, useMemo, useRef } from 'react'
import MessageElement from './MessageElement'
import { useConversationStore } from '../../hooks/conversations/useSelectedConversationStore'
import { useGetMessages } from '../../hooks/messages/useGetMessages'
import { useTranslation } from 'react-i18next'
import { useUserStore } from '../../hooks/users/useUserStore'
import { useMessageSettingsStore } from '../../hooks/messages/useMessageSettingsStore'
import Hirondelle from '../../ui/Hirondelle'
import { ToastContainer } from 'react-toastify'
import { useAtomValue } from 'jotai'
import { messageToEdit } from '../../hooks/atoms'
import ProgressBar from '../../ui/ProgressBar'

interface Props {
  responseChunks: string[]
  loading: boolean
  setIsProgressBarVisible: (val: boolean) => void
  isProgressBarVisible: boolean
}

const ChatBox = ({
  responseChunks,
  loading,
  isProgressBarVisible,
  setIsProgressBarVisible,
}: Props) => {
  const { t } = useTranslation()
  const { selectedConversation } = useConversationStore()
  const { selectedConvMode, setSelectedConvMode } = useMessageSettingsStore()
  const messageEdited = useAtomValue(messageToEdit)

  const { data: messages } = useGetMessages(selectedConversation?.id)
  const user = useUserStore((state) => state.user)
  const isCompleted = responseChunks && responseChunks.length > 0
  const timing = 60

  // === Refs ===
  const bottomOfMessagesRef = useRef<HTMLDivElement>(null)
  const scrollContainerRef = useRef<HTMLDivElement>(null)

  // === État logique pour le scroll auto ===
  const allowAutoScrollForGen = useRef(false) // autorisation d'autoscroll pendant cette génération
  const programmaticScroll = useRef(false) // empêche les scroll events auto d’être pris comme user
  const prevNewlineCount = useRef(0)
  const prevTextLength = useRef(0)
  const lastLoading = useRef(false)

  // === Buffer cumulé du message streamé ===
  const streamedText = useMemo(() => responseChunks.join(''), [responseChunks])
  const countNewlines = (s: string) => (s.match(/\n/g) || []).length

  // === Réinitialisation des flags au démarrage/fin de génération ===
  useEffect(() => {
    const wasLoading = lastLoading.current
    if (!wasLoading && loading) {
      // Début génération
      allowAutoScrollForGen.current = true
      programmaticScroll.current = false
      prevNewlineCount.current = countNewlines(streamedText)
      prevTextLength.current = streamedText.length
    } else if (wasLoading && !loading) {
      // Fin génération
      allowAutoScrollForGen.current = false
      programmaticScroll.current = false
      prevNewlineCount.current = 0
      prevTextLength.current = 0
    }
    lastLoading.current = loading
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [loading])

  // === Gestion du scroll manuel utilisateur ===
  useEffect(() => {
    const el = scrollContainerRef.current
    if (!el) return

    const handleScroll = () => {
      if (programmaticScroll.current) return
      if (!loading) return
      const threshold = 80
      const isNearBottom =
        el.scrollTop + el.clientHeight >= el.scrollHeight - threshold
      if (!isNearBottom) allowAutoScrollForGen.current = false
    }

    const handleUserIntent = () => {
      if (loading) allowAutoScrollForGen.current = false
    }

    el.addEventListener('scroll', handleScroll, { passive: true })
    el.addEventListener('wheel', handleUserIntent, { passive: true })
    el.addEventListener('touchmove', handleUserIntent, { passive: true })

    return () => {
      el.removeEventListener('scroll', handleScroll)
      el.removeEventListener('wheel', handleUserIntent)
      el.removeEventListener('touchmove', handleUserIntent)
    }
  }, [loading])

  // === Auto-scroll dynamique pendant la génération ===
  useEffect(() => {
    if (!loading) return

    const newLineCount = countNewlines(streamedText)
    const hasNewLine = newLineCount > prevNewlineCount.current
    const textGrew = streamedText.length > prevTextLength.current

    // Scroll si le texte grandit ou si nouvelle ligne, et si autorisé
    if (
      (hasNewLine || textGrew) &&
      allowAutoScrollForGen.current &&
      bottomOfMessagesRef.current
    ) {
      programmaticScroll.current = true
      // On attend un peu que le DOM se mette à jour avant de scroller
      setTimeout(() => {
        bottomOfMessagesRef.current?.scrollIntoView({ behavior: 'smooth' })
        // On ignore les scroll events déclenchés par ce scroll pendant 150ms
        setTimeout(() => {
          programmaticScroll.current = false
        }, 150)
      }, 10)
    }

    prevNewlineCount.current = newLineCount
    prevTextLength.current = streamedText.length
  }, [streamedText, loading])

  return (
    <>
      <ToastContainer className="fixed top-16 right-5 w-full z-70" />

      <div
        ref={scrollContainerRef}
        className="flex-grow overflow-y-auto flex flex-col relative min-h-[250px] py-4"
      >
        <div className="sticky top-2 z-10 px-3">
          <div className="flex w-full justify-start">
            <form id="conversation_mode">
              <select
                value={selectedConvMode}
                onChange={(e) => setSelectedConvMode(e.target.value)}
                aria-label={t('conversationMode')}
                name="conversation_mode"
                className="text-gray-900 text-sm rounded-md block w-full p-2.5 py-1.5 bg-white select-sidebar select-dropdown"
              >
                <option key="0" value="general">
                  {t('generalKnowledge')}
                </option>
                <option key="1" value="rag">
                  {t('rag')}
                </option>
              </select>
            </form>
          </div>
        </div>

        <div className="pt-2 2xl:pt-5 w-full px-5 max-w-3xl xl:max-w-4xl flex-grow flex flex-col relative mx-auto min-h-[100px]">
          <div className="fixed top-[40%] translate-y-[-50%] right-[0] md:right-[5%] lg:right-[8%] px-3">
            <Hirondelle />
          </div>

          {!selectedConversation || messages?.length === 0 ? (
            <div className="flex flex-col justify-between pt-2 xl:pt-5 flex-grow z-10">
              <div className="flex justify-start pb-10 max-w-5xl 2xl:max-w-6xl flex-grow">
                <div className="font-open font-thin leading-tight md:leading-snug gap-2 flex">
                  <p className="text-gradient-bnpp text-4xl 2xl:text-5xl">
                    {t('hello')},
                  </p>
                  <p className="text-gradient-bnpp text-4xl 2xl:text-5xl">
                    {user}
                  </p>
                </div>
              </div>
              <span className="text-gray-600 font-black font-open font-thin text-2xl md:text-2xl lg:text-2xl 2xl:text-3xl">
                {t('helpText')}
              </span>
            </div>
          ) : (
            selectedConversation &&
            messages?.map((message, index) => {
              const isLast = index === messages.length - 1
              const response = isLast ? responseChunks : []
              return (
                <div key={message.id} className="z-10">
                  <MessageElement
                    message={message}
                    timing={timing}
                    responseChunks={response}
                    loading={isLast ? loading : false}
                    isLast={isLast}
                    isCompleted={isCompleted}
                    isProgressBarVisible={isProgressBarVisible}
                    setIsProgressBarVisible={setIsProgressBarVisible}
                  />
                </div>
              )
            })
          )}

          <div ref={bottomOfMessagesRef} />
        </div>
      </div>
    </>
  )
}

export default ChatBox

```

```🎯 Objectif

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
```
