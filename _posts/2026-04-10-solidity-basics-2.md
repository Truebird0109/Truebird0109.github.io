---
layout: post
title: "Web3 시리즈 #7 — 솔리디티 기초 (하)"
date: 2026-04-10
categories: [web3-security]
tags: [solidity, modifier, event, require, error-handling]
---

## require / revert / assert — 에러 처리 삼총사

스마트 컨트랙트에서 "이 조건 아니면 실행 안 돼"를 표현하는 방법.

```solidity
function withdraw(uint amount) public {
    require(amount <= balances[msg.sender], "Insufficient balance");
    // 조건이 false면 여기서 멈추고 가스 환불
    
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

// assert: 절대 일어나면 안 되는 상황 체크. 가스 전부 소모
assert(totalSupply >= 0);  // 이게 false면 심각한 버그
```

**버그 바운티 팁**: `require` 누락은 매우 흔한 취약점이다. "이 함수에 require가 빠진 거 아닌가?" 항상 의심하자.

---

## modifier — 함수에 붙이는 조건

반복되는 검증 로직을 깔끔하게 빼는 방법.

```solidity
address public owner;

modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _;  // ← 원래 함수 코드가 여기서 실행됨
}

function setFee(uint _fee) public onlyOwner {
    fee = _fee;
}

function pause() public onlyOwner {
    paused = true;
}
```

`onlyOwner`를 한번 만들어놓으면 관리자 전용 함수마다 갖다 붙이면 된다.

### 보안 포인트

modifier를 **빼먹으면** 아무나 관리자 함수를 호출할 수 있다. 실제로 일어난 사건:

> 2017년 Parity Wallet 해킹: `initWallet()` 함수에 접근 제어가 없어서 해커가 자기를 owner로 등록 → **1.5억 달러 상당의 ETH 동결**

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

- 이벤트는 **트랜잭션 로그**에 저장됨
- **오프체인**(프론트엔드, 서버)에서 이벤트를 감지해 반응
- `indexed` 키워드: 해당 파라미터로 **필터링 검색** 가능
- 가스 비용이 storage보다 **훨씬 저렴**

### 보안 관점

이벤트는 보안 도구의 **눈**이다. 비정상적인 Transfer 이벤트를 모니터링해서 해킹을 탐지한다.

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
    // 인터페이스의 함수를 반드시 구현해야 함
    function transfer(address to, uint256 amount) external returns (bool) {
        // 구현...
    }
}
```

**인터페이스**는 "이 컨트랙트는 이런 함수들을 가지고 있어"라는 약속이다. Web3에서 컨트랙트끼리 소통할 때 필수.

---

## 외부 호출 — 컨트랙트가 컨트랙트를 부른다

```solidity
// 방법 1: 인터페이스로 호출
IERC20 token = IERC20(0x123...);
token.transfer(recipient, amount);

// 방법 2: 저수준 호출 (low-level call)
(bool success, bytes memory data) = target.call{value: 1 ether}(
    abi.encodeWithSignature("deposit()")
);
require(success, "Call failed");
```

### 보안 핵심: call의 위험성

`call`은 **가장 유연하지만 가장 위험한** 호출 방법이다.

- 실패해도 **revert되지 않음** — `success`를 직접 체크해야 함
- 보낸 쪽의 컨텍스트에서 임의 코드 실행 가능
- **Reentrancy Attack**의 진입점 (11편에서 자세히)

---

## fallback & receive — 돈을 받는 함수

```solidity
// ETH를 받을 때 자동 호출 (calldata 없이)
receive() external payable {
    emit Received(msg.sender, msg.value);
}

// 매칭되는 함수가 없을 때 호출
fallback() external payable {
    // ...
}
```

아무 데이터 없이 ETH만 보내면 → `receive()` 호출
존재하지 않는 함수를 호출하면 → `fallback()` 호출

이것도 보안에서 중요하다. Reentrancy Attack에서 핵심 역할을 한다.

---

## 정리

- `require`: 조건 검증의 기본. 누락 = 취약점
- `modifier`: 접근 제어의 핵심. `onlyOwner` 패턴
- `event`: 블록체인 로그. 모니터링의 기반
- 외부 호출(`call`): 유연하지만 위험. 항상 성공 여부 체크
- `receive`/`fallback`: ETH 수신 처리. Reentrancy의 무대

---

> **다음 편:** ERC-20 토큰이 뭔지, 그리고 토큰 표준이 왜 중요한지.
