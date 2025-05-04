---
title: Linux I/O 스케줄러 **BFQ** 완전 정리 – 구조, 장·단점, 튜닝
date: 2025-05-04
tags: [linux, kernel, io-scheduler, bfq, performance]
---

> **TL;DR**  
> BFQ(Budget Fair Queueing)는 _“가중치 기반 **예산(budget)**_”으로 디스크 대역폭을 공정하게 나누면서 GUI · 인터랙티브 앱에 **낮은 지연시간**을 보장하는 Linux 메인라인 블록-I/O 스케줄러다.  
> p70까지는 최강의 응답성을 주지만, **꼬리(p95+ latency)** 는 Kyber·mq-deadline보다 높아질 수 있다—트레이드오프를 이해하고 파라미터를 조정하면 된다.

---

## 1. BFQ 한눈에 보기

| 항목 | 내용 |
|------|------|
| **정식 명칭** | Budget Fair Queueing (block/blk-sq/bfq) |
| **메인라인 편입** | Linux v4.12 (2017) – CFQ 대체 |
| **서비스 모델** | • _bfq_queue_ 단위 독점 서비스<br>• 큐마다 _budget(섹터 수)_ 부여, 소진 시 교대 |
| **목표** | 1) 가중치-비례 대역폭<br>2) 인터랙티브·RT 작업 저지연<br>3) cgroup 계층 공유 |
| **알고리즘적 뿌리** | WF²Q+ / CFQ → 예산 개념 추가 |

---

## 2. 구조 – “예산”이 만드는 독점과 교대

1. **큐 생성**  
   - 프로세스·cgroup마다 `bfq_queue` 생성 → 각 큐에 **weight**(우선도) 부여  
2. **예산(budget) 할당**  
   - `max_budget`(기본 128 섹터) ± 피드백으로 동적 조절  
3. **독점 서비스**  
   - 큐가 예산을 다 쓰거나 타임아웃 되기 전까지 _디바이스 단독 사용_  
4. **피드백 루프**  
   - 순차 I/O면 예산 ↑, 랜덤·짧은 I/O면 예산 ↓ → 지연 최소화

---

## 3. 장점 👍

| 구분 | 상세 |
|------|------|
| **체감 응답성** | GUI, 터미널, 게임 로딩 등 p70 구간까지 지연 급감 |
| **가중치 기반 공정성** | `io.bfq.weight`로 **비율-공유 + 최소 보장** SLA |
| **HDD 친화적** | 랜덤 요청 병합 → _순차화_ → 스루풋 ~2× |
| **오디오/비디오 XRUN 방지** | soft-RT 스트림 자동 승격 (class RT) |
| **LTS 커널 유지보수** | 2024까지 Paolo Valente가 NVMe lock split 패치 진행 |

---

## 4. 단점 👎  – 꼬리(p95+) 지연이 길다

| 원인 | 설명 | 영향 |
|------|------|------|
| **예산 독점** | 쓰기 스트림이 큰 예산을 탕진할 때 | p95+ 지연 ms 단위 상승 |
| **전역 스핀락** | 동시 활성 큐가 많으면 lock 경합 → CPU 스톨 | NVMe multi-thread에서 p99 급등 |
| **HW QD 미활용** | 독점 큐 모델이라 HW 큐가 비어 있을 때도 있음 | IOPS 낮아짐 |

> **μs 급 p99** 가 필수(HFT, NVMe-oF, 5G RU 등)라면 **Kyber/mq-deadline** 또는 SPDK를 써야 한다.

---

## 5. 퍼센타일 분포 비교

| 구간 | 승자 | 이유 |
|------|------|------|
| **p0 ~ p70** | **BFQ** | 인터랙티브 I/O 즉시 서비스 |
| **p70 ~ p90** | 미세 우위 – BFQ | 예산 덕에 여전히 낮음 |
| **p90 ~ p99** | **Kyber·mq-deadline** | BFQ lock 경합 + 독점 tail   |

(실험 그래프는 [첨부 PNG](./latency_p0_p90.png) / [tail PNG](./latency_p90_p100.png) 참고)

---

## 6. 튜닝 레시피

```bash
# SSD / NVMe tail 완화 – 예산 축소
echo 64  > /sys/block/<dev>/queue/iosched/max_budget   # 기본 128

# 데스크톱 저지연 모드 유지
echo 1   > /sys/block/<dev>/queue/iosched/low_latency  # 기본 1

# 폭주 큐 rate-limit (cgroup v2)
echo "259:0 rbps=0 wbps=50M" > /sys/fs/cgroup/io.max   # 50 MiB/s write cap

# 대용량 배치 전용 서버 – 저지연 휴리스틱 오프
echo 0   > /sys/block/<dev>/queue/iosched/low_latency
