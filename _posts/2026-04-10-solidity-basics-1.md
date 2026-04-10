---
layout: post
title: "Web3 시리즈 #6 — 솔리디티 기초 (상)"
date: 2026-04-10
categories: [web3-security]
tags: [solidity, smart-contract, programming]
---

## 솔리디티(Solidity)란?

이더리움 스마트 컨트랙트를 작성하는 **프로그래밍 언어**다. JavaScript랑 문법이 좀 비슷해서 진입장벽이 낮다.

버그 바운티 하려면 **코드를 읽을 줄은 알아야** 하니까, 기초를 잡고 가자.

---

## 첫 번째 컨트랙트

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract HelloWeb3 {
    string public message;

    constructor() {
        message = "Hello, Web3!";
    }

    function setMessage(string memory _msg) public {
        message = _msg;
    }
}
```

한 줄씩 보자:

- `pragma solidity ^0.8.0` → 컴파일러 버전 지정 (0.8.x 사용)
- `contract HelloWeb3` → 클래스 같은 거. 컨트랙트 선언
- `string public message` → 상태 변수. **블록체인에 영구 저장**됨
- `constructor()` → 배포할 때 딱 한 번 실행
- `function setMessage(...)` → 외부에서 호출 가능한 함수

---

## 자료형

### 값 타입

```solidity
bool isHacker = true;          // 불리언
uint256 balance = 1000;         // 부호 없는 정수 (0 이상)
int256 temperature = -10;       // 부호 있는 정수
address owner = 0x742d...;      // 이더리움 주소 (20바이트)
bytes32 data = 0xabcd...;       // 고정 길이 바이트
```

**`uint256`이 기본이다.** Web3에서 가장 많이 보는 타입.

### 참조 타입

```solidity
string name = "Truebird";      // 문자열
uint[] numbers;                 // 동적 배열
mapping(address => uint) balances;  // 맵 (핵심!)
```

---

## mapping — 제일 중요한 자료구조

```solidity
mapping(address => uint256) public balances;
```

**키-값 저장소**다. Python의 딕셔너리, JS의 객체라고 보면 된다.

```solidity
balances[0xA...] = 100;   // A의 잔고 = 100
balances[0xB...] = 200;   // B의 잔고 = 200
```

ERC-20 토큰의 잔고, NFT의 소유자 등 **거의 모든 곳**에서 mapping을 쓴다.

주의: mapping은 **순회(iterate)가 불가능**하다. 어떤 키가 있는지 알 수 없다.

---

## 함수 가시성

```solidity
function pub() public { }     // 누구나 호출 가능
function ext() external { }   // 외부에서만 호출 가능 (컨트랙트 내부 X)
function int_() internal { }  // 이 컨트랙트 + 상속받은 컨트랙트만
function priv() private { }   // 이 컨트랙트만
```

| 가시성 | 외부 호출 | 내부 호출 | 상속 컨트랙트 |
|--------|-----------|-----------|--------------|
| public | ✅ | ✅ | ✅ |
| external | ✅ | ❌ | ❌ |
| internal | ❌ | ✅ | ✅ |
| private | ❌ | ✅ | ❌ |

### 보안 포인트

`public`으로 열어둔 함수는 **세상 누구나** 호출할 수 있다. 관리자 전용 함수를 `public`으로 열어뒀다면? **끝장이다.**

---

## 상태 변경 키워드

```solidity
function getBalance() public view returns (uint) {
    return balances[msg.sender];  // 읽기만 함
}

function getOwner() public pure returns (string memory) {
    return "Truebird";  // 상태 접근도 안 함
}

function deposit() public payable {
    balances[msg.sender] += msg.value;  // ETH를 받으면서 상태 변경
}
```

- **`view`**: 상태를 읽기만 함. 가스 무료 (외부 호출 시)
- **`pure`**: 상태에 아예 접근 안 함. 순수 계산
- **`payable`**: ETH를 받을 수 있음. 이거 안 붙이면 ETH 전송 시 에러

---

## msg 전역 변수

스마트 컨트랙트 안에서 항상 쓸 수 있는 특수 변수들:

```solidity
msg.sender   // 이 함수를 호출한 주소
msg.value    // 함께 보낸 ETH (wei 단위)
msg.data     // calldata 전체
block.timestamp  // 현재 블록 타임스탬프
block.number    // 현재 블록 번호
tx.origin    // 트랜잭션 최초 발신자 ← 보안 주의!
```

**`msg.sender` vs `tx.origin`** 차이는 나중에 보안 편에서 깊게 다룬다. 미리 스포하면: `tx.origin`으로 인증하면 **해킹당한다.**

---

## 정리

- 솔리디티 = 이더리움 스마트 컨트랙트 언어
- `uint256`, `address`, `mapping`이 핵심 타입
- 함수 가시성(public/external/internal/private) 잘못 쓰면 보안 구멍
- `msg.sender` = 호출자, `msg.value` = 보낸 ETH
- `payable` 안 붙이면 ETH 못 받음

---

> **다음 편:** modifier, event, require — 솔리디티의 나머지 핵심 문법과 에러 처리.
