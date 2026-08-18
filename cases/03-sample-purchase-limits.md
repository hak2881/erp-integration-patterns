# Sample Purchase Limits

**Trigger** · Request, on the cart path · **Enforcement point** · Before checkout completes · **Platform support** · None

## Context

In ingredient supply, samples are how a sale starts. A formulator requests small quantities of several materials, tests them, and comes back months later with a production order. Samples are priced far below cost — they are marketing, not revenue.

Which means they need a limit. Without one, the sample catalog becomes a cheap way to buy materials in bulk, one small order at a time.

## Problem

**The platform has no concept of a per-buyer purchase cap.** Inventory limits stock globally. Order limits cap a single order. Neither expresses "this company may take three sample units of this material, ever" or "…per quarter" — a rule about the buyer's history, not about this cart.

**The limit is a business rule that keeps moving.** Which products count as samples, how many are allowed, whether the window resets, whether it is per material or per category — all of it changed during the build and will change again.

**Enforcement has to happen before payment.** Rejecting after checkout means refunding a customer and explaining a rule they had no way to see. The cart is the last honest moment to say no.

**A validation call sits in the purchase path.** Whatever this service does, it does it while the buyer is waiting, and if it is down the correct behavior is not obvious — blocking all checkout because a limits service is unhealthy trades a small revenue leak for a total outage.

## Approach

**A separate service, on its own port, with its own lifecycle.** This is the most volatile logic in the group — the rule changes far more often than order sync or inventory sync do. Isolating it means a limits change deploys without touching anything that moves money into the ERP.

**The limit is evaluated against purchase history, not cart contents alone.** The question is "how many has this company already taken", which requires state the cart does not carry. The service owns that count rather than inferring it from what is in front of it.

**Configuration over code for the rule's parameters.** Quantities, eligible products, and window length are settings. The shape of the rule is code; the numbers in it are not.

**Fail-open was chosen deliberately, and it is a tradeoff not a default.** If the limits service cannot answer, checkout proceeds. The exposure is a bounded amount of under-priced product; the alternative is blocking every purchase — including full-price production orders — because a marketing guardrail is unavailable. That calculus would flip if samples were expensive or the cap were regulatory rather than commercial, and it is worth restating whenever either changes.

## What this case is really about

The interesting part is not the rule. It is recognizing that a piece of logic is *volatile* and giving it a boundary that matches its change rate — rather than filing it next to code that changes yearly and then redeploying an accounting integration every time marketing revises a sample allowance.

---

## 한국어 요약

원료 공급업에서 **샘플은 거래가 시작되는 지점**입니다. 제형 담당자가 원료 몇 가지를 소량으로 받아 테스트해보고, 몇 달 뒤에 양산 주문으로 돌아옵니다. 샘플 가격은 원가보다 한참 아래고, 매출이 아니라 마케팅 비용에 가깝습니다. 그래서 제한이 필요합니다. 안 걸어두면 샘플 카탈로그가 소량 주문을 반복해 원료를 싸게 사가는 경로가 됩니다.

**어려웠던 지점**

- **플랫폼에 "구매자별 상한"이라는 개념이 없습니다.** 재고는 전역으로 제한되고, 주문 제한은 주문 하나만 제한합니다. "이 회사는 이 원료 샘플을 평생 3개까지" 나 "분기당 3개까지"는 장바구니가 아니라 구매자의 이력에 대한 규칙이라 어느 쪽으로도 표현이 안 됩니다.
- **계속 바뀌는 비즈니스 규칙입니다.** 무엇을 샘플로 볼지, 몇 개까지 허용할지, 기간이 리셋되는지, 원료 단위인지 카테고리 단위인지. 구축하는 동안에도 전부 바뀌었고 앞으로도 바뀔 겁니다.
- **결제 전에 막아야 합니다.** 결제가 끝난 뒤에 거절하면 환불해주고, 고객이 알 방법도 없던 규칙을 설명해야 합니다. 장바구니가 정직하게 안 된다고 말할 수 있는 마지막 순간입니다.
- **검증 호출이 구매 경로에 놓입니다.** 구매자가 기다리는 동안 돌아가고, 장애가 났을 때 뭐가 맞는 동작인지도 딱 떨어지지 않습니다. 제한 서비스가 아프다고 결제를 전부 막으면 작은 매출 누수를 전면 장애와 바꾸는 셈이니까요.

**접근**

- **별도 서비스, 별도 포트, 별도 수명주기.** 이 그룹에서 가장 변덕스러운 로직입니다. 주문 동기화나 재고 동기화보다 훨씬 자주 바뀝니다. 떼어놓으면 제한 규칙을 고칠 때 ERP로 돈을 넣는 코드는 건드리지 않고 배포할 수 있습니다.
- **장바구니가 아니라 구매 이력을 보고 판정합니다.** 질문이 "이 회사가 지금까지 몇 개 받아갔나"라서 장바구니에는 없는 상태가 필요합니다. 눈앞의 카트에서 추론하는 대신 서비스가 그 카운트를 직접 갖고 있습니다.
- **규칙의 파라미터는 코드가 아니라 설정으로.** 수량, 대상 상품, 기간은 설정값입니다. 규칙의 모양은 코드로, 그 안에 들어가는 숫자는 설정으로 뒀습니다.
- **fail-open은 기본값이 아니라 일부러 고른 겁니다.** 제한 서비스가 답을 못 하면 결제를 그냥 진행시킵니다. 이때 감수하는 건 한정된 금액의 저가 상품이고, 반대로 갔다면 마케팅 가드레일 하나 때문에 정가 양산 주문까지 전부 막았을 겁니다. 샘플이 비싸지거나 상한이 상업적 기준이 아니라 규제 기준이 되면 이 계산은 뒤집히니, 그럴 때마다 다시 짚어봐야 합니다.

**이 케이스의 요점** — 재미있는 건 규칙 자체가 아닙니다. 어떤 로직이 유독 자주 바뀐다는 걸 알아보고 그 변경 주기에 맞춰 경계를 그은 판단이죠. 1년에 한 번 바뀌는 코드 옆에 두면, 마케팅이 샘플 허용량을 손볼 때마다 회계 연동을 다시 배포하게 됩니다.
