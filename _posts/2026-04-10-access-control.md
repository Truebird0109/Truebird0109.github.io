---
layout: post
title: "Web3 시리즈 #12 — Access Control 취약점"
date: 2026-04-10
categories: [web3-security]
tags: [access-control, tx-origin, authorization, security]
---

## 문지기가 없는 금고

Web2에서 인증 없이 관리자 페이지에 접근할 수 있다면 심각한 문제입니다.

Web3에서도 동일합니다. **관리자 전용 함수에 접근 제어가 없으면 치명적입니다.**

다만 파급력이 다릅니다. Web2는 데이터 유출, Web3는 **수백억 원이 증발**합니다.

---

## 패턴 1: 접근 제어 누락

```solidity
// ❌ 취약 — 누구든 호출 가능합니다
contract Vulnerable {
    address public owner;
    
    function setOwner(address _newOwner) public {
        owner = _newOwner;  // 누구나 owner를 변경할 수 있습니다!
    }
    
    function withdrawAll() public {
        payable(owner).transfer(address(this).balance);
    }
}
```

**공격 시나리오:**
1. 공격자가 `setOwner(공격자주소)` 호출
2. `withdrawAll()` 호출
3. 컨트랙트의 모든 ETH 탈취 완료

### 수정

```solidity
// ✅ 안전
modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _;
}

function setOwner(address _newOwner) public onlyOwner {
    owner = _newOwner;
}
```

---

## 패턴 2: tx.origin 인증의 함정

```solidity
// ❌ 취약
function transfer(address to, uint amount) public {
    require(tx.origin == owner, "Not owner");
    // ...
}
```

`tx.origin`은 **트랜잭션의 최초 발신자**입니다. `msg.sender`와의 차이는 무엇일까요?

```
사용자(EOA) → 악성 컨트랙트 → 피해자 컨트랙트
              msg.sender = 악성 컨트랙트
              tx.origin = 사용자(EOA) ← 여전히 원래 사용자입니다!
```

### 공격 시나리오

```solidity
contract PhishingAttack {
    VulnerableWallet wallet;
    address attacker;
    
    constructor(address _wallet) {
        wallet = VulnerableWallet(_wallet);
        attacker = msg.sender;
    }
    
    // 피해자가 이 컨트랙트와 상호작용하도록 유도합니다
    receive() external payable {
        // tx.origin은 피해자(owner)입니다!
        wallet.transfer(attacker, wallet.balance());
    }
}
```

1. 피해자(owner)가 악성 컨트랙트와 상호작용합니다
2. 악성 컨트랙트가 `wallet.transfer()`를 호출합니다
3. `tx.origin == owner`? → **True!** (최초 발신자가 owner이기 때문입니다)
4. 공격자에게 전액 전송됩니다

**결론: `tx.origin`으로 인증하지 마세요. 반드시 `msg.sender`를 사용하세요.**

---

## 패턴 3: 초기화 함수 취약점

```solidity
// ❌ 취약 — 프록시 패턴에서 자주 발생합니다
contract WalletLibrary {
    address public owner;
    
    // 누구나 호출 가능한 초기화 함수입니다!
    function initWallet(address _owner) public {
        owner = _owner;
    }
    
    function execute(address _to, uint _value, bytes memory _data) public onlyOwner {
        _to.call{value: _value}(_data);
    }
}
```

### 실제 사건: Parity Wallet Hack (2017)

1. `initWallet()`에 접근 제어가 없었습니다
2. 공격자가 `initWallet(공격자주소)` 호출 → owner 탈취
3. `execute()`로 **1.5억 달러 상당의 ETH 탈취**

이후 두 번째 사건 (같은 해):
- 다른 사용자가 남은 라이브러리 컨트랙트의 `initWallet()` 호출
- 자신을 owner로 만든 후 실수로 `selfdestruct` 실행
- 라이브러리가 파괴되면서 **5억 달러 상당의 ETH 영구 동결**

---

## 패턴 4: 역할 기반 접근 제어 실수

```solidity
// OpenZeppelin의 AccessControl 사용 예
contract Treasury is AccessControl {
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant WITHDRAWER_ROLE = keccak256("WITHDRAWER_ROLE");
    
    function withdraw(uint amount) public onlyRole(WITHDRAWER_ROLE) {
        // ...
    }
    
    // ❌ 역할 부여 함수 자체에 접근 제어가 없습니다
    function grantWithdrawer(address account) public {
        grantRole(WITHDRAWER_ROLE, account);  // 누구나 역할을 부여할 수 있습니다!
    }
}
```

역할(Role) 부여 함수 자체에 접근 제어가 없으면 역할 시스템 전체가 무의미해집니다.

---

## 체크리스트 — 버그 바운티에서 확인할 사항

```
□ onlyOwner/onlyAdmin이 누락된 함수가 있는가?
□ tx.origin을 인증에 사용하고 있는가?
□ initialize() 함수가 한 번만 호출되도록 보장하는가?
□ 역할 부여/제거 함수에 적절한 접근 제어가 있는가?
□ selfdestruct를 호출할 수 있는 함수가 보호되어 있는가?
□ 프록시의 admin 함수가 적절히 보호되어 있는가?
```

---

## 정리

- 접근 제어 누락 = **가장 단순하지만 가장 치명적인** 취약점
- `tx.origin` ≠ `msg.sender`. 인증에 `tx.origin` 사용은 절대 금물
- 초기화 함수는 **한 번만 호출 가능**해야 합니다
- 역할 관리 함수 자체에도 접근 제어가 필요합니다
- Parity 해킹: 접근 제어 하나가 빠져 **6.5억 달러 피해 발생**

---

> **다음 편:** Flash Loan Attack — 자본금 없이 수억 달러를 다루는 공격 기법을 살펴봅니다.
