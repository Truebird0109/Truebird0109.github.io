---
layout: post
title: Volatility로 메모리 덤프 분석하기
date: 2026-03-18
categories: [forensics]
tags: [forensics, memory, volatility]
---

메모리 포렌식은 RAM 덤프에서 실행 중인 프로세스, 네트워크 연결, 암호화 키 등을 추출하는 분야다.
Volatility는 가장 널리 쓰이는 메모리 분석 프레임워크로 `pslist`, `netscan`, `dumpfiles` 등의 플러그인을 제공한다.
CTF에서는 덤프 파일에서 플래그나 숨겨진 프로세스를 찾는 문제가 자주 출제된다.
