```import React, { useEffect, useMemo, useRef } from 'react'
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

  // Refs UI
  const bottomOfMessagesRef = useRef<HTMLDivElement>(null)
  const scrollContainerRef = useRef<HTMLDivElement>(null)

  // Refs logique par génération
  const generationId = useRef(0)                 // id de génération courant
  const allowAutoScrollForGen = useRef(false)    // autorisation latched pour la génération
  const programmaticScroll = useRef(false)       // ignore les scroll events causés par notre auto-scroll
  const prevNewlineCount = useRef(0)
  const prevTextLength = useRef(0)

  // Buffer cumulé du message streamé (les chunks sont des morceaux arbitraires)
  const streamedText = useMemo(() => responseChunks.join(''), [responseChunks])
  const countNewlines = (s: string) => (s.match(/\n/g) || []).length

  // Détecte front-edge du loading: false->true (nouvelle génération)
  const lastLoading = useRef(false)
  useEffect(() => {
    const wasLoading = lastLoading.current
    if (!wasLoading && loading) {
      // Début de la génération: réarme tout
      generationId.current += 1
      allowAutoScrollForGen.current = true
      programmaticScroll.current = false
      prevNewlineCount.current = countNewlines(streamedText)
      prevTextLength.current = streamedText.length
    } else if (wasLoading && !loading) {
      // Fin de la génération: bloque toute auto-tentative restante
      allowAutoScrollForGen.current = false
      programmaticScroll.current = false
      prevNewlineCount.current = 0
      prevTextLength.current = 0
    }
    lastLoading.current = loading
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [loading])

  // Gestion du scroll MANUEL sur le vrai conteneur scrollable
  useEffect(() => {
    const el = scrollContainerRef.current
    if (!el) return

    const handleScroll = () => {
      // Si c'est un scroll que NOUS venons de provoquer, on l'ignore
      if (programmaticScroll.current) return

      // Pendant la génération: dès que l’utilisateur s’éloigne du bas, on coupe l’autoscroll pour CETTE génération
      if (loading) {
        const threshold = 40
        const isNearBottom = el.scrollTop + el.clientHeight >= el.scrollHeight - threshold
        if (!isNearBottom) allowAutoScrollForGen.current = false
      }
    }

    // On coupe aussi si l’utilisateur touche molette ou tactile même si il est "près du bas"
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

  // Auto-scroll sur NOUVELLE LIGNE uniquement et uniquement si autorisé pour cette génération
  useEffect(() => {
    if (!loading) return // jamais d'autoscroll hors génération

    const newLineCount = countNewlines(streamedText)
    const hasNewLine = newLineCount > prevNewlineCount.current
    const textGrew = streamedText.length > prevTextLength.current

    // Conditions strictes: nouvelle ligne + texte qui grandit + autorisation en cours
    if (
      hasNewLine &&
      textGrew &&
      allowAutoScrollForGen.current &&
      bottomOfMessagesRef.current &&
      scrollContainerRef.current
    ) {
      // Marqueur "scroll programmatique" pour ignorer l'event 'scroll' généré
      programmaticScroll.current = true
      // Utilise rAF pour laisser le DOM mettre à jour les hauteurs avant le scroll
      requestAnimationFrame(() => {
        bottomOfMessagesRef.current!.scrollIntoView({ behavior: 'smooth' })
        // Le scroll smooth déclenchera des events; on enlève le flag après une courte fenêtre
        setTimeout(() => {
          programmaticScroll.current = false
        }, 350)
      })
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

        {
          !selectedConversation || messages?.length === 0 ? (
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
          )
        }

          <div ref={bottomOfMessagesRef} />
        </div>
      </div>
    </>
  )
}

export default ChatBox
```
