---
layout: post
title: "Web3 시리즈 #13 — Flash Loan Attack"
date: 2026-04-10
categories: [web3-security]
tags: [flash-loan, defi, exploit, security]
---

## Flash Loan이란?

> 1 트랜잭션 안에서 **담보 없이** 수억 달러를 빌리고, **같은 트랜잭션에서 상환하는** 대출

현실 세계에서는 불가능한 개념입니다. 은행에 "1초만 100억 빌려주세요, 바로 갚겠습니다"라고 하면 쫓겨날 것입니다.

블록체인에서는 가능합니다. 이유는 다음과 같습니다.

**트랜잭션은 원자적(atomic)**이기 때문입니다.
- 트랜잭션의 모든 단계가 성공 → 전부 적용됩니다
- 하나라도 실패 → **전부 되돌립니다** (상환하지 못하면 대출 자체가 취소됩니다)

```solidity
// Flash Loan 기본 구조 (Aave 스타일)
function flashLoan(uint amount) external {
    uint balanceBefore = token.balanceOf(address(this));
    
    token.transfer(msg.sender, amount);      // 1. 자금을 빌려줍니다
    
    IFlashBorrower(msg.sender).onFlashLoan(amount);  // 2. 차용자가 활용합니다
    
    // 3. 상환 여부를 확인합니다
    require(
        token.balanceOf(address(this)) >= balanceBefore + fee,
        "Flash loan not repaid"
    );
}
```

---

## 합법적인 활용 사례

Flash Loan 자체는 중립적인 도구입니다.

### 차익 거래 (Arbitrage)

```
Uniswap: 1 ETH = 3000 USDC
SushiSwap: 1 ETH = 3050 USDC

1. Flash Loan으로 300만 USDC 차용
2. Uniswap에서 ETH 1000개 구매 (300만 USDC 사용)
3. SushiSwap에서 ETH 1000개 판매 (305만 USDC 수령)
4. Flash Loan 상환 (300만 + 수수료)
5. 이익: 약 5만 USDC (수수료 차감 후)
```

**초기 자본 없이** 차익 거래가 가능합니다. 시장 효율성에도 기여합니다.

### 부채 리파이낸싱

```
1. Flash Loan으로 기존 대출금 차용
2. A 프로토콜의 고금리 대출 상환
3. 담보 회수
4. B 프로토콜에 담보 재예치 후 저금리 대출
5. Flash Loan 상환
```

---

## Flash Loan Attack의 원리

문제는 Flash Loan을 **가격 조작**에 활용할 때입니다.

### 전형적인 공격 흐름

```
1. Flash Loan으로 대량의 토큰 A를 차용합니다
2. DEX에서 토큰 A를 대량 매도하여 가격을 폭락시킵니다
3. 폭락한 가격을 참조하는 프로토콜에서 이익을 취합니다
4. 이익을 다시 토큰 A로 전환합니다
5. Flash Loan을 상환합니다
6. 차익을 확보합니다
```

**이 모든 과정이 1 트랜잭션 안에서 완료됩니다.**

---

## 실제 사례: bZx 해킹 (2020)

DeFi 최초의 Flash Loan 공격 중 하나입니다.

```
1. dYdX에서 10,000 ETH Flash Loan 차용
2. 5,500 ETH를 Compound에 담보로 예치하고 112 WBTC 대출
3. 1,300 ETH로 Uniswap에서 WBTC 대량 매수 (유동성이 낮아 가격 급등)
4. bZx에서 나머지 ETH로 WBTC 숏 포지션 개설
5. WBTC 가격이 인위적으로 높아진 상태에서 bZx의 마진 계산 조작
6. Compound에서 차용한 WBTC를 높은 가격에 Uniswap에서 매도
7. Flash Loan 상환
8. 순이익: 약 35만 달러
```

핵심: **bZx가 Uniswap의 현물 가격(spot price)을 오라클로 사용**했기 때문에 가능했습니다.

---

## 왜 방어가 어려운가?

### 1. 자본 장벽이 없습니다

전통 금융에서 시장을 조작하려면 **막대한 자본**이 필요합니다. Flash Loan은 **초기 자본 없이** 가능합니다.

### 2. 한 트랜잭션 = 한 블록

모든 조작이 **하나의 블록** 안에서 완료되므로, 모니터링 시스템이 **사전에 감지할 수 없습니다.**

### 3. 리스크 대비 보상

실패 시: 가스비만 소모됩니다. 성공 시: 수백만 달러의 이익. **비대칭적인 리스크 구조**입니다.

---

## 방어 전략

### 1. TWAP 오라클 사용

```
❌ 현물 가격 (spot price) = 단일 블록의 가격
   → Flash Loan으로 조작 가능합니다

✅ TWAP (Time-Weighted Average Price) = 다수 블록의 가중 평균 가격
   → 단일 블록만으로는 조작이 불가능합니다
```

### 2. Chainlink 같은 외부 오라클 사용

```
❌ 온체인 DEX 가격 참조
   → 유동성 조작으로 가격 변경 가능합니다

✅ Chainlink 오라클 = 오프체인 데이터 집계
   → 여러 소스의 가격을 중앙값으로 제공합니다
```

### 3. 동일 블록 내 가격 변동 제한

```solidity
// 이전 블록과 가격 차이가 임계값을 초과하면 거부합니다
require(
    priceDeviation < MAX_DEVIATION, 
    "Price manipulation detected"
);
```

---

## 버그 바운티에서 확인할 사항

다음 질문들을 검토하세요:

1. **이 프로토콜은 가격을 어디서 가져오는가?** (온체인 DEX? Chainlink?)
2. **Flash Loan으로 이 가격 소스를 조작할 수 있는가?**
3. **조작된 가격 상태에서 이 프로토콜을 이용하면 어떤 이익을 취할 수 있는가?**
4. **TWAP을 사용하더라도 기간이 충분히 긴가?**

---

## 정리

- Flash Loan = 1 트랜잭션 내 무담보 대출 (상환 실패 시 전부 취소)
- 그 자체로는 중립적인 도구. 가격 조작에 활용되면 **공격 수단**
- 핵심: 온체인 현물 가격을 오라클로 사용하면 **반드시 조작당합니다**
- 방어: TWAP, Chainlink, 가격 변동 제한
- 초기 자본 없음, 최소 리스크, 최대 보상 → 공격자에게 매력적인 기법

---

> **다음 편:** Oracle Manipulation — 가격 데이터 조작을 더 깊이 살펴봅니다.
