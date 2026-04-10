---
layout: post
title: "Web3 시리즈 #13 — Flash Loan Attack"
date: 2026-04-10
categories: [web3-security]
tags: [flash-loan, defi, exploit, security]
---

## Flash Loan이 뭔데?

> 1 트랜잭션 안에서 **담보 없이** 수억 달러를 빌리고, **같은 트랜잭션에서 갚는** 대출

현실에서는 불가능하다. 은행에서 "1초만 100억 빌려주세요, 바로 갚을게요" 하면 쫓겨남.

블록체인에서는 가능하다. 왜?

**트랜잭션은 원자적(atomic)**이기 때문이다.
- 트랜잭션의 모든 단계가 성공하면 → 전부 적용
- 하나라도 실패하면 → **전부 되돌림** (갚지 못하면 빌린 것도 취소)

```solidity
// Flash Loan 기본 구조 (Aave 스타일)
function flashLoan(uint amount) external {
    uint balanceBefore = token.balanceOf(address(this));
    
    token.transfer(msg.sender, amount);      // 1. 돈 빌려줌
    
    IFlashBorrower(msg.sender).onFlashLoan(amount);  // 2. 빌린 놈이 뭔가 함
    
    // 3. 갚았는지 확인
    require(
        token.balanceOf(address(this)) >= balanceBefore + fee,
        "Flash loan not repaid"
    );
}
```

---

## 합법적인 활용

Flash Loan 자체는 나쁜 게 아니다.

### 차익 거래 (Arbitrage)

```
Uniswap: 1 ETH = 3000 USDC
SushiSwap: 1 ETH = 3050 USDC

1. Flash Loan으로 300만 USDC 빌림
2. Uniswap에서 1000 ETH 구매 (300만 USDC)
3. SushiSwap에서 1000 ETH 판매 (305만 USDC)
4. Flash Loan 갚기 (300만 + 수수료)
5. 이익: ~5만 USDC (수수료 제외)
```

**자본금 0**으로 차익 거래. 시장 효율성에 기여한다.

### 부채 리파이낸싱

```
1. Flash Loan으로 대출금 빌림
2. A 프로토콜의 비싼 대출 상환
3. 담보 회수
4. B 프로토콜에 담보 예치, 더 싸게 대출
5. Flash Loan 갚기
```

---

## Flash Loan Attack의 원리

문제는 Flash Loan을 **가격 조작**에 쓸 때다.

### 전형적인 공격 흐름

```
1. Flash Loan으로 대량의 토큰 A를 빌림
2. DEX에서 토큰 A를 한꺼번에 팔아서 가격 폭락시킴
3. 가격이 폭락한 상태에서, 그 가격을 참조하는 프로토콜에서 이득을 봄
4. 이득을 다시 토큰 A로 바꿈
5. Flash Loan 갚기
6. 차익 챙기고 도주
```

**모든 게 1 트랜잭션 안에서 일어난다.**

---

## 실제 사례: bZx 해킹 (2020)

DeFi 최초의 Flash Loan 공격 중 하나.

```
1. dYdX에서 10,000 ETH Flash Loan
2. 5,500 ETH를 Compound에 담보로 넣고 112 WBTC 대출
3. 1,300 ETH로 Uniswap에서 WBTC 가격을 끌어올림 (유동성이 적어서 가능)
4. bZx에서 나머지 ETH로 WBTC 숏 포지션 오픈
5. WBTC 가격이 인위적으로 높아진 상태에서 bZx의 마진 계산 조작
6. Compound에서 빌린 WBTC를 높은 가격에 Uniswap에서 판매
7. Flash Loan 상환
8. 순이익: ~35만 달러
```

핵심: **bZx가 Uniswap의 현물 가격(spot price)을 오라클로 사용**했기 때문에 가능했다.

---

## 왜 방어가 어려운가?

### 1. 자본 장벽이 없다

전통 금융에서 시장을 조작하려면 **엄청난 자본**이 필요하다. Flash Loan은 **자본금 0**으로 가능.

### 2. 한 트랜잭션 = 한 블록

모든 조작이 **하나의 블록** 안에서 끝나므로, 모니터링 시스템이 **사전에 감지할 수 없다.**

### 3. 완벽한 범죄

실패하면 가스비만 날아감. 성공하면 수백만 달러 이득. **리스크 대비 보상이 미친 수준.**

---

## 방어 전략

### 1. TWAP 오라클 사용

```
❌ 현물 가격 (spot price) = 1 블록의 가격
   → Flash Loan으로 조작 가능

✅ TWAP (Time-Weighted Average Price) = 여러 블록의 평균 가격
   → 1 블록만으로는 조작 불가
```

### 2. Chainlink 같은 외부 오라클

```
❌ 온체인 DEX 가격 참조
   → 유동성 조작으로 가격 변경 가능

✅ Chainlink 오라클 = 오프체인 데이터 집계
   → 여러 소스의 가격을 중앙값으로 제공
```

### 3. 같은 블록 내 가격 변동 제한

```solidity
// 이전 블록과 가격 차이가 너무 크면 거부
require(
    priceDeviation < MAX_DEVIATION, 
    "Price manipulation detected"
);
```

---

## 버그 바운티 팁

이 질문들을 던져라:

1. **이 프로토콜은 가격을 어디서 가져오나?** (온체인 DEX? Chainlink?)
2. **Flash Loan으로 이 가격 소스를 조작할 수 있나?**
3. **가격 조작된 상태에서 이 프로토콜을 이용하면 어떤 이득을 볼 수 있나?**
4. **TWAP을 쓰더라도 기간이 충분히 긴가?**

---

## 정리

- Flash Loan = 1 트랜잭션 내 무담보 대출 (갚지 못하면 전부 취소)
- 자체로는 합법. 가격 조작에 쓰면 **공격**
- 핵심: 온체인 현물 가격을 오라클로 쓰면 **반드시 조작당함**
- 방어: TWAP, Chainlink, 가격 변동 제한
- 자본금 0, 리스크 최소, 보상 최대 → 해커의 꿈

---

> **다음 편:** Oracle Manipulation — 가격 데이터 조작의 기술을 더 깊이 파본다.
