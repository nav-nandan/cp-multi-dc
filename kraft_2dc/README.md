# 2 DC KRaft topologies
docker-compose base to test CP KRaft for 2 DC setup across the following configurations:

---

## 2 DC Stretched Cluster with KRaft in Isolated mode using Dynamic Controller Configuration

![](https://github.com/nav-nandan/cp-multi-dc/blob/main/kraft_2dc/2dc_stretched_kraft_isolated.png)

### Resiliency Matrix
| Scenario                            | Partition-level roles & ISR behavior                                                                              | Observation for clients (connecting to cluster)                      |
| ----------------------------------- | ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Baseline – all healthy              | Each partition: 1 leader on one of {B1–B6}, remaining 5 replicas as followers; ISR=6.                             | Fully available; local R/W from both DCs; acks=all works everywhere. |
| Full DC1 outage (B1–B3, C1–C3 down) | DC1 data & voters gone. DC2 still has RF=3 (3 replicas/partition), but controller quorum is lost (only 2 voters). | Cluster is down until ≥3 voters are re‑established (via new controllers in DC2/3rd DC). Clients will see increasing failures once leaders/ISR need to change.|
| Full DC2 outage (B4–B6, C4–C6 down) | DC2 offline; DC1 still has RF=3 (3 replicas/partition); controller quorum OK in DC1.                              | Available but single‑DC. DC2‑side clients must fail over to DC1; DC1 continues serving R/W with ISR=3. |

---

## 2 DC Multi Region Cluster (MRC) with KRaft in Isolated mode using Dynamic Controller Configuration

![](https://github.com/nav-nandan/cp-multi-dc/blob/main/kraft_2dc/2dc_mrc_kraft_isolated.png)

### Resiliency Matrix
| Scenario                            | Partition-level roles & ISR behavior                                                                                           | Observation for clients (connecting to cluster)                      |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| Baseline – all healthy              | Each partition: leader ∈ {B1,B2,B4,B5}, 3 followers (remaining syncs), observers on B3,B6. ISR=4 (2/DC), O=2.                  | Fully available; local R/W from both DCs; acks=all works everywhere. |
| Full DC1 outage (B1–B3, C1–C3 down) | Same quorum loss as ID 10, plus loss of all DC1 replicas. DC2 still has sync+observer replicas, but controller quorum is gone. | Cluster is down until ≥3 voters are re‑established (via new controllers in DC2/3rd DC). Clients will see increasing failures once leaders/ISR need to change. |
| Full DC2 outage (B4–B6, C4–C6 down) | Quorum OK on DC1; DC1 is now the only DC. Partitions run entirely on B1–B3 with leaders/followers/observers there. ISR≈2–3.    | Available but single‑DC. DC2‑side clients must fail over to DC1; DC1 continues serving R/W with ISR=3. |

---

## 2 DC Multi Region Cluster (MRC) with KRaft in Combined mode using Dynamic Controller Configuration

![](https://github.com/nav-nandan/cp-multi-dc/blob/main/kraft_2dc/2dc_mrc_kraft_combined.png)

### Resiliency Matrix
| Scenario                                 | Partition-level roles & ISR behavior                                                                                                                                                        | Observation for clients (connecting to cluster)                      |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Baseline – all healthy                   | Each partition: leader on one of {N1,N2,N4,N5}, 3 followers (remaining syncs), observers on N3,N6. ISR=4 (2/DC), +2 observers.                                                              | Fully available; local R/W in both DCs; low latency; acks=all OK. |
| All 3 combined nodes in DC1 down (N1–N3) | All DC1 replicas gone. Each partition now has only DC2 replicas on N4,N5,N6 (2 sync +1 observer). But only 2 voters left → controller quorum lost: ISR/leaders can’t change safely anymore. | Cluster is not healthy: existing leaders still serve for a while, but any leader/ISR change will get stuck; must restore ≥3 voters. |
| All 3 combined nodes in DC2 down (N4–N6) | All DC2 replicas gone. Each partition now runs only on DC1 (2 sync +1 observer across N1,N2,N3). Quorum OK on DC1.                                                                          | Available but single‑DC. DC2 clients must fail over to DC1 endpoints; R/W continues normally from DC1. |

---

## Further Testing Scenarios

| Scenario - Isolated Mode                                |
| ------------------------------------------------------- |
| Single broker down (B1)                                 |
| Two brokers down in DC1 (B1,B2)                         |
| Two brokers down, split (B1 in DC1, B4 in DC2)          |
| All brokers in DC1 down (B1–B3)                         |
| All brokers in DC2 down (B4–B6)                         |
| Single controller down in DC1 (C1)                      |
| Single controller down in DC2 (C4)                      |
| Both DC2 voters down (C4,C5) – all DC2 controllers down |
| All DC2 controllers down (C4–C6)                        |
| All DC1 controllers down (C1–C3) – DC2 controllers up   |
| Any combo with ≥3 voters AND ≥3 ISR/partition           |
| Any combo with <3 voters (controller quorum broken)     |

| Scenario - Combined Mode                              |
| ----------------------------------------------------- |
| Single combined node down (DC1) (N1)                  |
| Single combined node down (DC2) (N4)                  |
| Two nodes down in same DC1 (N1, N2)                   |
| Two nodes down in same DC2 (N4, N5)                   |
| Mixed: one node down in each DC (N1,N4)               |
| Any pattern with ≥3 voters AND ≥3 ISR/partition       |
| Any pattern with <3 voters (controller quorum broken) |
