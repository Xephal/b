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

  // Refs
  const bottomOfMessagesRef = useRef<HTMLDivElement>(null)
  const scrollContainerRef = useRef<HTMLDivElement>(null)

  // Flags et compteurs pour la génération courante
  const autoScrollLocked = useRef(false) // user a scrolled up pendant cette génération
  const ignoreScrollUntil = useRef<number | null>(null) // ignore les scroll events auto
  const prevNewlineCount = useRef(0) // nombre de \n déjà vus
  const prevTextLength = useRef(0) // taille précédente du texte

  // Buffer cumulé du message streamé
  const streamedText = useMemo(() => responseChunks.join(''), [responseChunks])

  const countNewlines = (s: string) => (s.match(/\n/g) || []).length

  // Réinitialisation à chaque génération
  useEffect(() => {
    if (loading) {
      autoScrollLocked.current = false
      prevNewlineCount.current = countNewlines(streamedText)
      prevTextLength.current = streamedText.length
    } else {
      // Fin de génération : bloquer temporairement tout scroll automatique
      autoScrollLocked.current = true
      ignoreScrollUntil.current = null

```
