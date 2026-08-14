# Scheduled Inventory Sync

**Trigger** · Schedule, twice daily · **Direction** · ERP → storefront · **Shape** · A CLI job, not a server

## Context

Stock exists in two places. The ERP knows what is physically in the warehouse, because it is also what the offline sales team, the production floor, and the accountants use. The storefront shows a number to customers.

Only one of those can be the truth. It is the ERP.

## Problem

**SKU is the join key, and the two systems disagree about it.** The ERP has its own material codes, built up over years, with formatting conventions that predate the store. The storefront has variant SKUs. Matching them is not a string comparison, and any material that fails to match is a product whose stock silently never updates.

**Overstating stock is much worse than understating it.** Selling something that isn't there means calling a manufacturing customer to tell them their raw material shipment is delayed. Showing less than you have costs a sale you might not have made anyway. The sync's error behavior should be asymmetric.

**Two updates a day is a deliberate staleness budget.** Real-time inventory sync from an ERP means either polling it constantly or asking it to push, and this ERP does neither comfortably. For materials with long production lead times and B2B order cycles measured in weeks, a stock figure that is a few hours old is accurate enough — and admitting that in the design is better than pretending otherwise with a fragile near-real-time pipeline.

## Approach

```mermaid
flowchart LR
    CR[Scheduler<br/>00:00 · 17:00 KST] --> J[Sync job]
    J --> E[(ERP<br/>stock quantities)]
    E --> M{SKU match}
    M -->|matched| U[Storefront<br/>inventory update]
    M -->|unmatched| L[(Report unmatched<br/>as job output)]
    U --> S[Storefront]
```

**It is a scheduled job, not a service.** No HTTP surface, no idle pods, no health endpoint pretending to mean something. It starts, reconciles, reports, and exits. The failure mode of a job that didn't run is far easier to detect than a service that is up but has quietly stopped doing its work.

**Unmatched SKUs are output, not warnings.** Every material the job could not map is reported as part of the run's result. An unmatched SKU is a product page showing a stale number to a customer — it is a data problem someone has to fix in one of the two systems, and it stays visible until they do.

**Run times chosen around the business, not around load.** Just after midnight to reflect the previous day's closing position, and late afternoon to catch the day's movements before evening ordering. Both are outside the window when warehouse staff are actively adjusting quantities, so the job reads a settled figure rather than a moving one.

**The job is idempotent by construction.** It reads current ERP quantities and sets storefront levels to match. It does not apply deltas. A re-run after a partial failure converges to the same state instead of double-counting, which means "just run it again" is always a safe response.

## What I would revisit

The twice-daily cadence is right for the ordering rhythm, but the window between runs is exactly where an oversell happens — a large order in the morning isn't reflected until evening. Consuming order webhooks to decrement locally between full syncs would close it, with the scheduled run continuing to serve as the authoritative reconciliation. That is the standard pattern: a fast approximate path plus a slow authoritative one, where the slow path is the arbiter.

---

## 한국어 요약

재고는 두 곳에 존재합니다. ERP는 창고에 물리적으로 뭐가 있는지 압니다 — 오프라인 영업팀·생산 현장·회계가 같은 걸 쓰기 때문입니다. 스토어프론트는 고객에게 숫자를 보여줍니다. **둘 중 하나만 진실일 수 있고, 그건 ERP입니다.**

**어려웠던 지점**

- **SKU가 조인 키인데 두 시스템의 의견이 다릅니다.** ERP에는 스토어보다 오래된 표기 관행으로 수년간 쌓인 자체 품목 코드가 있고, 스토어프론트에는 변형(variant) SKU가 있습니다. 매칭은 문자열 비교가 아니며, 매칭에 실패한 품목은 **재고가 조용히 영원히 갱신되지 않는 상품**이 됩니다.
- **재고 과다 표시가 과소 표시보다 훨씬 나쁩니다.** 없는 걸 팔면 제조사 고객에게 전화해서 원료 입고가 늦어진다고 말해야 합니다. 있는 것보다 적게 보여주면 어차피 안 났을지도 모르는 판매 하나를 잃습니다. 동기화의 에러 동작은 **비대칭이어야** 합니다.
- **하루 두 번은 의도한 staleness 예산입니다.** ERP에서 실시간 재고 동기화를 하려면 계속 폴링하거나 ERP가 푸시해야 하는데, 이 ERP는 둘 다 편하게 해주지 않습니다. 생산 리드타임이 길고 B2B 주문 주기가 주 단위인 원료라면 **몇 시간 된 재고 숫자로 충분히 정확**하고, 이걸 설계에서 인정하는 편이 취약한 준실시간 파이프라인으로 아닌 척하는 것보다 낫습니다.

**접근**

- **서비스가 아니라 스케줄 잡입니다.** HTTP 표면도, 유휴 파드도, 의미 있는 척하는 헬스 엔드포인트도 없습니다. 시작 → 대사 → 리포트 → 종료. **안 돈 잡**의 장애는 **떠 있지만 조용히 일을 안 하는 서비스**보다 훨씬 감지하기 쉽습니다.
- **매칭 실패 SKU는 경고가 아니라 산출물입니다.** 매핑하지 못한 모든 품목을 실행 결과의 일부로 보고합니다. 매칭 안 된 SKU는 곧 고객에게 낡은 숫자를 보여주는 상품 페이지이고, 두 시스템 중 한쪽에서 사람이 고쳐야 하는 데이터 문제이며, 고칠 때까지 계속 보입니다.
- **실행 시각은 부하가 아니라 업무 리듬 기준.** 자정 직후(전일 마감 반영)와 늦은 오후(당일 변동을 저녁 주문 전에 반영). 둘 다 창고 인력이 수량을 활발히 조정하는 시간대 밖이라, 움직이는 값이 아니라 **정착된 값**을 읽습니다.
- **구조적으로 멱등합니다.** ERP의 현재 수량을 읽어 스토어프론트 레벨을 그 값으로 **설정**합니다. 증감(delta)을 적용하지 않습니다. 부분 실패 후 재실행해도 중복 반영 없이 같은 상태로 수렴하므로, "그냥 다시 돌리세요"가 항상 안전한 답이 됩니다.

**다시 한다면** — 하루 두 번이라는 주기는 주문 리듬에 맞지만, **실행 사이의 간격이 정확히 oversell이 발생하는 구간**입니다. 오전의 대량 주문이 저녁까지 반영되지 않습니다. 주문 웹훅을 받아 전체 동기화 사이에 로컬 차감을 하면 이 구멍이 닫히고, 스케줄 실행은 계속 권위 있는 대사 역할을 합니다. 흔한 패턴입니다 — **빠른 근사 경로 + 느린 권위 경로, 판정은 느린 쪽이.**
