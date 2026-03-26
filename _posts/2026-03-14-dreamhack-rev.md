---
layout: post
title: Dreamhack rev-basic 풀이
date: 2026-03-14
categories: [wargame]
tags: [wargame, dreamhack, reversing]
---

Dreamhack의 rev-basic은 리버싱 입문자를 위한 문제로 바이너리를 분석해 올바른 입력값을 찾아야 한다.
Ghidra나 IDA Free로 디컴파일하면 문자열 비교 로직을 쉽게 파악할 수 있다.
조건 분기를 따라가며 각 바이트를 역추적하면 플래그를 얻을 수 있다.
