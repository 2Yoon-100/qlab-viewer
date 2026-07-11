# QLab 데이터셋 — 데이터 사전 v2.0

> 코드명 **QLab**. 청산 규칙 **A-last**. 전 데이터 `data_type="backtest"`. 포워드(실시간)는 해시봉인 이후 분만 `data_type="forward"`. **백테스트를 실시간 실적으로 오인 금지.**
> v1(B-first·미보정)은 `product_v1/`에 동결. 변경 상세는 `CHANGELOG.md`.

## 변경 이력 (v1 → v2.0)
- **청산 규칙**: B-first(첫매수가+50% 단일) → **A-last(SMA200 조건부: 200일선 아래 +50%/위 +100%, 기준가=마지막매수가)**
- **유니버스 보정**: KR 상폐주 포함(Tier1+), S&P500 point-in-time(Tier1); NASDAQ·코인은 미보정(Tier3 경고)
- **스키마**: +`tier`, +`rule_version`, `exit_reason` 세분(6종)
- 사유: 생존편향 실측 보정 + A/B 검증에서 A-last 우월. (전체 → `CHANGELOG.md`)

## 파일
- `qlab_v2_{KR,US-SP500,US-NASDAQ,MAJOR,ALT}.parquet` — 시장/티어별 일별(day≤365)
- `qlab_v2_all.parquet` — 통합본
- `qlab_v2_sample_100.csv` — 무료 샘플(시장비례·품질분포 반영)

## 컬럼 (25)
| 컬럼 | 정의 | 비고 |
|---|---|---|
| `signal_id` | 신호 ID (`QL2-YYYYMMDD-티커`) | v2 접두 QL2 |
| `market` | 시장 | KR/US-SP500/US-NASDAQ/MAJOR/ALT |
| `ticker` | 종목·심볼 | |
| `tier` | **신뢰 등급** | Tier1+/Tier1/Tier3 (아래) |
| `entry_date` | 진입일(day 0) | |
| `day` | 진입 후 경과 거래일 | 주말/휴장 행 없음(정상) |
| `date` | 거래일 달력일 | |
| `close` | 종가 | 시장별 정규장 마감 |
| `return_from_entry_pct` | 진입 종가 대비 수익률 | day0 = 0.00 |
| `rsi_bucket`…`kimchi_bucket` | DNA 9종 **버킷**(원본 미노출) | 진입 시점 |
| `quality_score` / `quality_bucket` | 크로스마켓 품질점수(0–100) / 구간 | 진입 시점 산출 |
| `fear_bucket` | 공포 진입등급 | 적극/선별/관망 |
| `exit_reason` | **청산 사유(세분)** | TP50 / TP50_추세꺾임 / TP100 / 만기손절 / 상폐 / 보유중 |
| `total_holding_days` | 보유 거래일 = min(청산, 365) | |
| `rule_version` | 채점 규칙 버전 | `A-last-v1` |
| `data_type` | 데이터 구분 | backtest / forward |

## 신뢰 등급 (Tier) — 생존편향 기준
| Tier | 대상 | 상태 | 근거 |
|---|---|---|---|
| **Tier 1+** | KR (상폐주 포함) | **완전 보정(실측)** | 상폐 553/605 포함. 보정 후 승률 −2%p, 상폐손실 KOSDAQ −60.6%/KOSPI −17.5% |
| **Tier 1** | US-SP500 (point-in-time) | **제거(측정됨)** | ever-member 재백테스트. current 대비 승률 −3%p. 잔존: 미반영 161(시세소실) |
| **Tier 3** | US-NASDAQ | **미보정 — 경고** | 현재 상장목록. 누락 30~40% 추정, **성과 상방(낙관) 편향**. 절대치는 상한선 |
| **Tier 3** | 코인 (MAJOR/ALT) | **미보정 — 경고** | 거래지원종료 코인 제외 → 생존편향 존재 |

- **외삽 금지**: S&P500 제거는 피인수 다수(실패 아님) — NASDAQ 상폐(실패 위주)와 편향 방향 다름. 3%p를 NASDAQ에 외삽 금지.
- **포워드**: 신호 시점 해시봉인 → 종목 상폐돼도 과거 스냅샷 불변 + `detect_delisting()`로 상폐 청산 기록 → **생존편향 구조적 0**.

## 청산 규칙 (A-last)
매 거래일 종가 평가 (기준가 = 마지막 매수가 entryPrice):
1. `종가 < SMA200 AND 종가 ≥ 기준가×1.5` → **TP50** (직전 거래일 종가 > SMA200이면 **TP50_추세꺾임**)
2. `종가 > SMA200 AND 종가 ≥ 기준가×2.0` → **TP100**
3. 365일 경과 미청산 → **만기손절**  ·  보유 중 상폐 → **상폐**(상폐일 최종가)

## 시장별 기준 시각 · available_at
| 시장 | 일봉 종가 기준 | available_at |
|---|---|---|
| KR | 15:30 KST (KRX 마감) | 당일 15:30 KST 이후 |
| US | 16:00 ET (미 마감) | 익일 05~06시 KST 이후 |
| 코인 | 00:00 UTC | 당일 09:00 KST 이후 |

- **available_at = 룩어헤드 없음**: 위 시각 이전 미확정 → 그 이후에만 신호 유효.
- KST 15:00 조기 알림은 **한국 전용**(미국·코인 미적용).

## 자본 기준
- 거래당 **100만원 고정 · 단리 · 재투자 없음**. 손익 = 100만 × 수익률.
