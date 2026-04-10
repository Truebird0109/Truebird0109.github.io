---
layout: post
title: "Web3 시리즈 #7 — 솔리디티 기초 (하)"
date: 2026-04-10
categories: [web3-security]
tags: [solidity, modifier, event, require, error-handling]
---

## require / revert / assert — 에러 처리 삼총사

스마트 컨트랙트에서 "이 조건을 만족하지 않으면 실행하지 않는다"를 표현하는 방법입니다.

```solidity
function withdraw(uint amount) public {
    require(amount <= balances[msg.sender], "Insufficient balance");
    // 조건이 false이면 여기서 실행을 멈추고 가스를 환불합니다
    
    balances[msg.sender] -= amount;
    payable(msg.sender).transfer(amount);
}
```

### 차이점

```solidity
// require: 입력값/조건 검증. 실패 시 남은 가스 환불
require(amount > 0, "Amount must be positive");

// revert: 조건문 안에서 수동으로 되돌리기
if (amount == 0) {
    revert("Amount must be positive");
}

// assert: 절대 발생해서는 안 되는 상황을 점검. 가스 전부 소모
assert(totalSupply >= 0);  // 이것이 false라면 심각한 버그입니다
```

**버그 바운티 팁**: `require` 누락은 매우 흔한 취약점입니다. "이 함수에 require가 빠진 것은 아닐까?" 항상 의심하는 습관이 필요합니다.

---

## modifier — 함수에 붙이는 조건

반복되는 검증 로직을 깔끔하게 분리하는 방법입니다.

```solidity
address public owner;

modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _;  // ← 원래 함수의 코드가 여기서 실행됩니다
}

function setFee(uint _fee) public onlyOwner {
    fee = _fee;
}

function pause() public onlyOwner {
    paused = true;
}
```

`onlyOwner`를 한 번 정의해두면 관리자 전용 함수마다 붙여서 사용할 수 있습니다.

### 보안 포인트

modifier를 **누락하면** 누구든 관리자 함수를 호출할 수 있습니다. 실제 발생한 사건:

> 2017년 Parity Wallet 해킹: `initWallet()` 함수에 접근 제어가 없어 공격자가 자신을 owner로 등록 → **1.5억 달러 상당의 ETH 동결**

---

## event — 블록체인의 로그

```solidity
event Transfer(address indexed from, address indexed to, uint256 value);

function transfer(address _to, uint256 _value) public {
    require(balances[msg.sender] >= _value);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    
    emit Transfer(msg.sender, _to, _value);  // 이벤트 발생
}
```

- 이벤트는 **트랜잭션 로그**에 저장됩니다
- **오프체인**(프론트엔드, 서버)에서 이벤트를 감지해 반응할 수 있습니다
- `indexed` 키워드: 해당 파라미터로 **필터링 검색**이 가능합니다
- 스토리지보다 **가스 비용이 훨씬 저렴**합니다

### 보안 관점

이벤트는 보안 모니터링 도구의 **눈**입니다. 비정상적인 Transfer 이벤트를 감지해 해킹을 조기에 탐지할 수 있습니다.

---

## 상속과 인터페이스

```solidity
// 인터페이스 - 함수 시그니처만 정의
interface IERC20 {
    function transfer(address to, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

// 상속
contract MyToken is IERC20 {
    // 인터페이스의 함수를 반드시 구현해야 합니다
    function transfer(address to, uint256 amount) external returns (bool) {
        // 구현...
    }
}
```

**인터페이스**는 "이 컨트랙트는 이런 함수들을 가지고 있다"는 약속입니다. Web3에서 컨트랙트끼리 통신할 때 필수적으로 사용됩니다.

---

## 외부 호출 — 컨트랙트가 컨트랙트를 호출합니다

```solidity
// 방법 1: 인터페이스를 통한 호출
IERC20 token = IERC20(0x123...);
token.transfer(recipient, amount);

// 방법 2: 저수준 호출 (low-level call)
(bool success, bytes memory data) = target.call{value: 1 ether}(
    abi.encodeWithSignature("deposit()")
);
require(success, "Call failed");
```

### 보안 핵심: call의 위험성

`call`은 **가장 유연하지만 가장 위험한** 호출 방법입니다.

- 실패해도 **revert되지 않습니다** — `success` 여부를 직접 확인해야 합니다
- 임의 코드가 실행될 수 있습니다
- **Reentrancy Attack**의 주요 진입점입니다 (#11편에서 자세히 다룹니다)

---

## fallback & receive — ETH를 수신하는 함수

```solidity
// 데이터 없이 ETH만 받을 때 자동 호출
receive() external payable {
    emit Received(msg.sender, msg.value);
}

// 매칭되는 함수가 없을 때 호출
fallback() external payable {
    // ...
}
```

데이터 없이 ETH만 전송하면 → `receive()` 호출
존재하지 않는 함수를 호출하면 → `fallback()` 호출

이 두 함수는 보안에서 중요합니다. Reentrancy Attack에서 핵심적인 역할을 합니다.

---

## 정리

- `require`: 조건 검증의 기본. 누락되면 취약점이 됩니다
- `modifier`: 접근 제어의 핵심. `onlyOwner` 패턴
- `event`: 블록체인 로그. 보안 모니터링의 기반입니다
- 외부 호출(`call`): 유연하지만 위험합니다. 항상 성공 여부를 확인하세요
- `receive`/`fallback`: ETH 수신 처리. Reentrancy의 무대입니다

---

> **다음 편:** ERC-20 토큰이 무엇인지, 그리고 토큰 표준이 왜 중요한지 알아보겠습니다.
