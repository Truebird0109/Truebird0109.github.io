---
layout: post
title: Linux 권한 상승 기법 정리
date: 2026-03-16
categories: [penetration-testing]
tags: [pentest, privesc, linux]
---

초기 침투 후 권한 상승(Privilege Escalation)은 root 또는 관리자 권한을 획득하는 과정이다.
SUID 바이너리, sudo 미스컨피그, cron job, writable /etc/passwd 등이 주요 벡터로 활용된다.
LinPEAS, LinEnum 같은 자동화 스크립트로 취약점 후보를 빠르게 열거할 수 있다.
