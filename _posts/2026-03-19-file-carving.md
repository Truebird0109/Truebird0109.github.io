---
layout: post
title: File Carving으로 삭제된 파일 복구하기
date: 2026-03-19
categories: [forensics]
tags: [forensics, file-carving, recovery]
---

File Carving은 파일 시스템 정보 없이 파일 시그니처(매직 바이트)를 기반으로 파일을 복구하는 기법이다.
JPEG는 `FF D8 FF`, ZIP은 `50 4B 03 04` 같은 헤더로 시작 위치를 찾는다.
Foremost, Scalpel 같은 도구를 자주 사용하며 CTF 포렌식 문제에서 단골로 등장한다.
