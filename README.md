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

  // Refs
  const bottomOfMessagesRef = useRef<HTMLDivElement>(null)
  const scrollContainerRef = useRef<HTMLDivElement>(null)

  // Flags de génération
  const autoScrollLocked = useRef(false)
  const ignoreScrollUntil = useRef<number | null>(null)
  const prevNewlineCount = useRef(0)

  // Buffer cumulé
  const streamedText = useMemo(() => responseChunks.join(''), [responseChunks])

  // Compteur de retours à la ligne
  const countNewlines = (s: string) => (s.match(/\n/g) || []).length

  // Quand une génération démarre / se termine
  useEffect(() => {
    if (loading) {
      autoScrollLocked.current = false
      prevNewlineCount.current = countNewlines(streamedText)
    } else {
      // génération terminée
      ignoreScrollUntil.current = null
      prevNewlineCount.current = 0
      // on ne fait rien si le user avait scrollé (autoScrollLocked=true)
      // => pas de scroll final forcé
      autoScrollLocked.current = false
    }
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [loading])

  // Écoute le scroll manuel
  useEffect(() => {
    const el = scrollContainerRef.current
    if (!el) return

    const handleScroll = () => {
      // ignorer le scroll déclenché par scrollIntoView le temps de l’animation
      if (ignoreScrollUntil.current && Date.now() < ignoreScrollUntil.current) {
        return
      }

      const threshold = 40
      const isNearBottom =
        el.scrollTop + el.clientHeight >= el.scrollHeight - threshold

      // si on n’est pas près du bas pendant la génération → user a scrollé
      if (!isNearBottom && loading) {
        autoScrollLocked.current = true
      }
    }

    el.addEventListener('scroll', handleScroll, { passive: true })
    return () => el.removeEventListener('scroll', handleScroll)
  }, [loading])

  // Auto-scroll sur nouvelle ligne uniquement pendant la génération
  useEffect(() => {
    if (!loading) return

    const newCount = countNewlines(streamedText)
    const hasNewLine = newCount > prevNewlineCount.current

    if (
      hasNewLine &&
      !autoScrollLocked.current &&
      bottomOfMessagesRef.current &&
      scrollContainerRef.current
    ) {
      bottomOfMessagesRef.current.scrollIntoView({ behavior: 'smooth' })
      ignoreScrollUntil.current = Date.now() + 400
    }

    prevNewlineCount.current = newCount
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
