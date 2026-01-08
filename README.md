```
private function filterUsageDataByWeekday(array $data, ?int $weekday): array
{
    if ($weekday === null) {
        return $data;
    }

    $filtered = [];
    foreach ($data as $key => $_) {
        $filtered[$key] = [];
    }

    foreach ($data['date'] as $index => $date) {
        $dayOfWeek = (int) (new \DateTimeImmutable($date))->format('N'); // 1 = lundi

        if ($dayOfWeek !== $weekday) {
            continue;
        }

        foreach ($data as $key => $values) {
            $filtered[$key][] = $values[$index];
        }
    }

    return $filtered;
}
```

```
$data = [
    'date' => $dates,
    'messagesPerDay' => $messagesPerDay,
    'activeUsersPerDay' => $activeUsersPerDay,
    'avgResponseTimePerDay' => $avgResponseTimePerDay,
    'connectionPerDay' => $connectionPerDay,
    'conversationPerDay' => $conversationPerDay,
];

$data = $this->filterUsageDataByWeekday($data, $weekday);

return $data;

```
