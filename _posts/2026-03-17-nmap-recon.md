---
layout: post
title: Nmap을 활용한 네트워크 정찰
date: 2026-03-17
categories: [penetration-testing]
tags: [pentest, nmap, recon]
---

침투 테스트의 첫 단계는 정찰(Reconnaissance)로, Nmap을 통해 열린 포트와 서비스 버전을 파악한다.
`nmap -sV -sC -oN scan.txt <target>` 명령으로 기본 스크립트 스캔과 버전 탐지를 동시에 수행할 수 있다.
수집된 정보를 바탕으로 CVE를 검색하거나 다음 공격 벡터를 결정한다.
