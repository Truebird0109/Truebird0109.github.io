---
layout: post
title: SQL Injection 정리
date: 2026-03-26
categories: [web-hacking]
tags: [sql-injection, web, sqli]
---

## 서론

SQL Injection이란 웹 애플리케이션이 데이터베이스에 보내는 쿼리를 공격자가 조작할 수 있도록 만드는 보안 취약점입니다.

이 공격을 통해 공격자가 얻을 수 있는 것은 아래와 같습니다.

- 다른 사용자의 민감한 정보 같은 민감한 데이터 열람
- 데이터 수정/삭제 및 애플리케이션의 동작 변경
- 서버 탈취, 백도어 설치, 서비스 거부 공격 등

---

## 본론

SQL Injection의 탐지 방법은 **수동 탐지 방법**과 **자동화 공격** 두 가지 방법이 있습니다.

### 수동 탐지 방법

1. `'` 입력 후 에러 발생 여부 확인
2. `OR 1=1`, `OR 1=2` 같은 boolean 조건 삽입 후 응답 비교
3. 타임 딜레이 기반 페이로드 사용 (`SLEEP(5)` 등)
4. OAST를 활용해 외부 요청 여부 확인

### 자동화 탐지 방법

Burp Scanner 또는 SQLmap을 활용할 수 있습니다. 하지만 실제 상황에서는 SQLmap을 사용 못하는 경우가 있으니 유의합시다.

---

SQL Injection은 대부분 `SELECT WHERE` 문에서 발생하지만 아래와 같은 위치에서도 발생할 수 있습니다.

- `UPDATE` 문의 `SET`, `WHERE`
- `INSERT` 문의 값
- `SELECT` 문의 테이블 / 컬럼 이름
- `ORDER BY` 절

---

## 공격 기법

### 1. 숨겨진 데이터 열람

```sql
SELECT * FROM products WHERE category = '사용자 입력' AND released = 1
```

이런 코드로 데이터를 조회하는 타겟이 있다고 가정합니다. 입력 가능한 건 `category` 뿐입니다.

이때 category에 `'--` 를 입력할 수 있다면 어떻게 될까요?

```sql
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
```

`Gifts` 뒤에 있던 `AND` 부터 쭉 주석 처리가 되어 숨겨진 데이터를 확인할 수 있게 됩니다.

---

### 2. 로그인 우회

```sql
SELECT * FROM users WHERE username = '입력 ID' AND password = '입력 Password'
```

ID, Password를 입력받는 곳이 있다고 가정합니다. 공격자가 어떻게든 계정의 ID를 알아냈다면 `ADMIN'--` 를 입력하는 순간:

```sql
SELECT * FROM users WHERE username = 'ADMIN'--' AND password = ''
```

Password를 입력하지 않고도 로그인할 수 있을지도 모릅니다. 단, Server 측에서 검증하는 방식이라면 실패할 것입니다.

---

### 3. UNION을 이용한 다른 테이블 정보 열람

UNION은 두 개 이상의 SELECT 쿼리 결과를 합쳐 하나의 결과로 반환하는 명령어입니다.
UNION의 사용 조건은 **동일한 컬럼 수**, **데이터 타입 일치**입니다.

**Union SQLi 탐지 방법:**

**[1]** `'` 입력 시 오류가 뜨면 SQL Injection 가능성이 높음

**[2]** 컬럼 개수 확인

```sql
' ORDER BY 1--   -- 에러가 나는 지점을 찾으면 됩니다
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```

**[3]** 문자열 출력 가능한 컬럼 찾기

```sql
' UNION SELECT 'a',NULL,NULL--
' UNION SELECT NULL,'a',NULL--
' UNION SELECT NULL,NULL,'a'--
```

페이지에 `a` 가 보이면 해당 컬럼은 문자열 출력이 가능합니다.

**[4]** 민감한 데이터 추출

```sql
' UNION SELECT username, password FROM users--
```

---

### 4. Blind SQL Injection

응답에 쿼리 결과나 오류가 직접 표시되지 않는 경우에 사용할 수 있는 방법입니다.

우회적인 접근 방법:

1. 조건에 따라 응답이 달라지게 만들기
2. `SLEEP()` 을 사용하여 응답 시간 차이로 판단
3. OAST 기법을 활용한 네트워크 요청 유도

---

### 5. Second-order SQL Injection

첫 요청에서는 입력을 DB에 안전하게 저장했지만, 나중에 저장된 값을 사용하는 쿼리에 취약점이 존재할 때 사용할 수 있는 방법입니다.

해당 공격에 대해서는 더 공부해본 뒤 추가 글을 작성하겠습니다.

---

## 결론

데이터베이스 별로 공격 방법도 다르고 DB 구성 방법에 따라 공격 가능성 여부가 갈립니다. 자동화된 툴도 좋지만 직접 찾아보는 연습을 해보는 것도 좋은 것 같습니다.
