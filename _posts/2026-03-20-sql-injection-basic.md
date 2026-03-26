---
layout: post
title: SQL Injection 기초 정리
date: 2026-03-20
categories: [web-hacking]
tags: [sqli, web, injection]
---

SQL Injection은 사용자 입력값이 쿼리에 그대로 삽입될 때 발생하는 취약점이다.
`' OR '1'='1` 같은 페이로드로 인증을 우회하거나 데이터를 탈취할 수 있다.
방어는 Prepared Statement 사용과 입력값 검증으로 이루어진다.
