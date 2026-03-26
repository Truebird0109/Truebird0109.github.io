---
layout: post
title: XSS(Cross-Site Scripting) 공격 원리
date: 2026-03-21
categories: [web-hacking]
tags: [xss, web, javascript]
---

XSS는 공격자가 악성 스크립트를 웹 페이지에 삽입해 피해자 브라우저에서 실행시키는 공격이다.
Reflected, Stored, DOM-based 세 가지 유형으로 나뉘며 쿠키 탈취, 세션 하이재킹에 활용된다.
방어는 출력 시 HTML 엔티티 인코딩과 CSP(Content Security Policy) 설정으로 한다.
