protected function getUsageData(
    ?\DateTimeInterface $fromDate,
    ?\DateTimeInterface $toDate,
    bool $forChart = false
): array {
    $messagesPerDay        = $this->messageRepository->totalMessagesPerDay($fromDate, $toDate);
    $activeUsersPerDay     = $this->messageRepository->activeUsersPerDay($fromDate, $toDate);
    $avgResponseTimePerDay = $this->messageRepository->averageResponseTimePerDay($fromDate, $toDate);

    $data = [
        'messagesPerDay'        => $messagesPerDay,
        'activeUsersPerDay'     => $activeUsersPerDay,
        'avgResponseTimePerDay' => $avgResponseTimePerDay,
    ];

    // Initialisation du tableau de retour par jour
    $return = [];

    if ($fromDate && $toDate) {
        $current = clone $fromDate;

        while ($current <= $toDate) {
            $day = $current->format('d/m/Y');

            $return[$day] = [
                'messagesPerDay'        => 0,
                'activeUsersPerDay'     => 0,
                'avgResponseTimePerDay' => 0,
                'conversationPerDay'    => 0, // requis par prepareData()
                'messagesPerConversation' => 0, // requis aussi
            ];

            $current = (clone $current)->modify('+1 day');
        }
    }

    if ($forChart) {
        return $this->prepareDataForChart($data, $return);
    }

    return $this->prepareData($data, $return);
}
