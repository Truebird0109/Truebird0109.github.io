---
layout: post
title: "Web3 시리즈 #14 — Oracle Manipulation"
date: 2026-04-10
categories: [web3-security]
tags: [oracle, price-manipulation, chainlink, twap, security]
---

## 오라클 문제 — 블록체인의 근본적 한계

블록체인은 **외부 세계를 인식하지 못합니다.**

ETH의 달러 가격, 오늘의 날씨, 축구 경기 결과... 블록체인 밖의 데이터를 **스스로 가져올 수 없습니다.**

이 외부 데이터를 블록체인에 전달하는 역할을 하는 것이 **오라클(Oracle)**입니다.

```
외부 세계 (ETH = $3000)
      ↓ 오라클
블록체인 (ETH = $3000임을 인식)
      ↓
DeFi 프로토콜 (이 가격을 기반으로 대출/청산/스왑 계산)
```

**오라클이 잘못된 가격을 제공하면? DeFi 전체가 잘못된 가격으로 동작합니다.**

---

## 오라클의 종류

### 1. 온체인 오라클 (DEX 가격 참조)

```solidity
// Uniswap V2 풀에서 현재 가격 조회
(uint reserve0, uint reserve1, ) = pair.getReserves();
uint price = reserve1 * 1e18 / reserve0;
```

**장점**: 탈중앙화, 추가 비용 없음  
**단점**: **Flash Loan으로 조작 가능** ← 이것이 핵심 문제입니다

### 2. 오프체인 오라클 (Chainlink 등)

```solidity
// Chainlink에서 가격 조회
(, int price, , , ) = priceFeed.latestRoundData();
```

**장점**: 조작이 극히 어려움, 여러 소스를 집계  
**단점**: 중앙화 요소 존재, 업데이트 지연 가능성

### 3. TWAP 오라클

```
Time-Weighted Average Price
= 일정 시간 동안의 가중 평균 가격

30분 TWAP = 최근 30분간 각 블록 가격의 가중 평균
```

**장점**: 단일 블록 조작으로는 변경이 어려움  
**단점**: 가격 반영이 느립니다

---

## Oracle Manipulation 공격

### 기본 공격 구조

```
1. 대량의 자금 확보 (Flash Loan 활용)
2. DEX에서 대량 매수/매도 → 온체인 가격 조작
3. 조작된 가격을 참조하는 프로토콜에서 이익 취득
4. 원상 복구
5. Flash Loan 상환 + 차익 확보
```

### 구체적 시나리오: 렌딩 프로토콜 공격

```
정상 상태:
  토큰 X 가격 = $1
  담보율 = 70%
  
공격:
1. Flash Loan으로 토큰 X 대량 매수 → 가격 $1 → $10
2. 보유 중인 토큰 X를 렌딩 프로토콜에 담보로 예치
3. 담보 실제 가치 = ($1 × 수량) but 프로토콜은 ($10 × 수량)으로 계산
4. 과대 평가된 담보 가치의 70%만큼 다른 자산 대출
5. 대출 자산을 확보하고 Flash Loan 상환
6. 토큰 X 가격 원상 복구 → 담보 가치 폭락 → 대출 미상환
```

---

## 실제 공격 사례

### Harvest Finance (2020) — 3,400만 달러

```
1. Flash Loan으로 USDC/USDT 대량 확보
2. Curve에서 USDC → USDT 대량 스왑 (USDC 가격 하락)
3. Harvest의 USDC 풀에 저렴해진 USDC로 입금 (더 많은 fToken 수령)
4. Curve에서 USDT → USDC 스왑 (USDC 가격 복구)
5. Harvest에서 fToken 출금 (더 많은 USDC 수령)
6. 7회 반복 → 3,400만 달러 이익
```

### Mango Markets (2022) — 1.14억 달러

```
1. 두 계정으로 MNGO 선물 롱/숏 포지션 동시 개설
2. 현물 시장에서 MNGO 대량 매수 → 가격 급등
3. 오라클이 상승한 가격 반영
4. 롱 포지션의 미실현 이익 폭증
5. 그 이익을 담보로 Mango에서 모든 자산 대출
6. 결과: 1.14억 달러 탈취 (공격자는 이후 체포됨)
```

---

## 취약한 오라클 패턴 식별

### ❌ 위험한 패턴

```solidity
// 1. 단일 DEX 현물 가격 참조
uint price = reserveB / reserveA;  // Flash Loan으로 즉시 조작 가능

// 2. 단일 오라클 소스 의존
uint price = singleOracle.getPrice();  // 해당 소스 장애 시 문제 발생

// 3. 이상치 검증 없는 오라클 업데이트
function updatePrice(uint _price) external onlyOracle {
    price = _price;  // 이상값 검증 없음
}

// 4. 오래된 가격 데이터 사용
(, int price, , uint updatedAt, ) = feed.latestRoundData();
// updatedAt 검증 없음 → 수 시간 전 가격일 수 있습니다
```

### ✅ 안전한 패턴

```solidity
// 1. Chainlink + 신선도 검증
(, int price, , uint updatedAt, ) = priceFeed.latestRoundData();
require(price > 0, "Invalid price");
require(block.timestamp - updatedAt < MAX_STALENESS, "Stale price");

// 2. 다중 오라클 소스의 중앙값 사용
uint price1 = chainlink.getPrice();
uint price2 = uniswapTWAP.getPrice();
uint price3 = bandProtocol.getPrice();
uint price = median(price1, price2, price3);

// 3. 가격 변동 제한
uint newPrice = oracle.getPrice();
uint deviation = abs(newPrice - lastPrice) * 100 / lastPrice;
require(deviation < MAX_DEVIATION_PERCENT, "Price deviation too high");
```

---

## 버그 바운티 체크리스트

```
□ 가격 소스가 어디인가? (온체인 DEX? Chainlink? 자체 오라클?)
□ 가격이 1 트랜잭션 내에서 조작 가능한가?
□ 오라클 데이터의 신선도(staleness)를 검증하는가?
□ 단일 소스 의존인가, 다중 소스인가?
□ 급격한 가격 변동을 제한하는 로직이 있는가?
□ TWAP을 사용한다면, 기간이 충분히 긴가? (최소 30분 이상 권장)
□ 유동성이 낮은 토큰의 가격을 참조하고 있지 않은가?
```

---

## 정리

- 오라클 = 블록체인에 외부 데이터를 전달하는 브릿지
- 온체인 현물 가격(DEX spot) = Flash Loan으로 조작 가능
- Chainlink = 가장 안전하지만, staleness 검증이 필수
- TWAP = 단일 블록 조작 방어, 기간 설정이 중요
- 가격 조작 + Flash Loan = DeFi 해킹의 핵심 콤보

---

> **다음 편 (최종편):** Web3 버그 바운티 로드맵 — 여기까지 학습했다면 시작할 준비가 되었습니다.
