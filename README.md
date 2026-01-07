
    public function activeUsersPerDay($from = null, $to = null): array
    {
        $qb = $this->createQueryBuilder('m')
            ->select('COUNT(DISTINCT conversation.user) AS count')
            ->addSelect("DATE_FORMAT(m.createdAt, '%d/%m/%Y') AS day")
            ->join('m.conversation', 'conversation')
            ->join('conversation.user', 'user')
            ->andWhere('user.isExcludedForStats = false')
            ->groupBy('day')
            ->orderBy('day', 'ASC');

        if ($from !== null) {
            $qb->andWhere('m.createdAt >= :from')
            ->setParameter('from', $from);
        }

        if ($to !== null) {
            $qb->andWhere('m.createdAt <= :to')
            ->setParameter('to', $to);
        }

        return $qb->getQuery()->getResult();
    }

    public function averageResponseTimePerDay($from = null, $to = null): array
    {
        $qb = $this->createQueryBuilder('m')
            ->select('AVG(m.timeToAnswer) AS count')
            ->addSelect("DATE_FORMAT(m.createdAt, '%d/%m/%Y') AS day")
            ->join('m.conversation', 'conversation')
            ->join('conversation.user', 'user')
            ->andWhere('m.timeToAnswer IS NOT NULL')
            ->andWhere('user.isExcludedForStats = false')
            ->groupBy('day')
            ->orderBy('day', 'ASC');

        if ($from !== null) {
            $qb->andWhere('m.createdAt >= :from')
            ->setParameter('from', $from);
        }

        if ($to !== null) {
            $qb->andWhere('m.createdAt <= :to')
            ->setParameter('to', $to);
        }

        return $qb->getQuery()->getResult();
    }

