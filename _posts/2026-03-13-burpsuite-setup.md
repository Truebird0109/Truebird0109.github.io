---
layout: post
title: Burp Suite 초기 세팅 가이드
date: 2026-03-13
categories: [etc]
tags: [tools, burpsuite, setup]
---

Burp Suite는 웹 해킹에서 필수적인 프록시 도구로 HTTP 요청/응답을 가로채 분석하고 수정할 수 있다.
설치 후 Proxy 리스너를 `127.0.0.1:8080`으로 설정하고 브라우저 프록시와 연결한 뒤 CA 인증서를 설치해야 HTTPS도 분석된다.
Community Edition으로도 기본적인 Intercept, Repeater, Intruder 기능을 충분히 활용할 수 있다.
