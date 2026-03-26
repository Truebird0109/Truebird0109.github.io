---
layout: post
title: "[TEST] Markdown 렌더링 테스트"
date: 2026-03-01
categories: [etc]
tags: [test, markdown]
---

# H1 제목
## H2 제목
### H3 제목
#### H4 제목

---

## 텍스트 스타일

일반 텍스트입니다.

**굵게(bold)** 텍스트입니다.

*기울임(italic)* 텍스트입니다.

***굵게+기울임*** 텍스트입니다.

~~취소선~~ 텍스트입니다.

`인라인 코드` 텍스트입니다.

---

## 링크 & 이미지

[링크 텍스트](https://example.com)

![이미지 테스트]({{ "/assets/images/jjack.jpg" | relative_url }})

---

## 목록

**순서 없는 목록:**
- 항목 1
- 항목 2
  - 중첩 항목 2-1
  - 중첩 항목 2-2
- 항목 3

**순서 있는 목록:**
1. 첫 번째
2. 두 번째
3. 세 번째

---

## 인용구

> 이것은 인용구(blockquote)입니다.
> 여러 줄도 가능합니다.

>> 중첩 인용구도 됩니다.

---

## 코드 블록

**인라인 코드:** `nmap -sV 192.168.1.1`

**bash:**
```bash
#!/bin/bash
echo "Hello, Truebird!"
nmap -sV -sC -oN scan.txt 192.168.1.1
```

**python:**
```python
def exploit(target):
    payload = "' OR '1'='1"
    response = requests.get(target + "?id=" + payload)
    return response.text

print(exploit("http://example.com/login"))
```

**javascript:**
```javascript
const payload = "<script>alert('XSS')</script>";
document.getElementById("input").value = payload;
```

---

## 표(Table)

| 취약점 | 위험도 | 방어 방법 |
|--------|--------|-----------|
| SQL Injection | 높음 | Prepared Statement |
| XSS | 중간 | 출력 인코딩 |
| CSRF | 중간 | CSRF Token |
| LFI | 높음 | 경로 검증 |

---

## 수평선

---

***

___

---

## 줄바꿈

첫 번째 줄입니다.
두 번째 줄입니다. (줄바꿈 없음)

첫 번째 문단입니다.

두 번째 문단입니다. (빈 줄로 문단 분리)

---

## 체크리스트

- [x] SQL Injection 공부
- [x] XSS 공부
- [ ] CSRF 공부
- [ ] SSRF 공부
