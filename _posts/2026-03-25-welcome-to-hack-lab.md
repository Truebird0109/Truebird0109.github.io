---
layout: post
title: "블로그를 시작합니다 - Hack Lab 오픈!"
date: 2026-03-25
categories: [general]
tags: [blog, intro, hacking]
comments: true
---

## 블로그를 시작합니다!

안녕하세요, **Truebird** 입니다.

해킹과 보안을 공부하면서 배운 내용들을 기록하기 위해 이 블로그를 만들었습니다. 앞으로 이곳에 다양한 주제의 글을 올릴 예정입니다.

### 다룰 주제들

- **Web Hacking** - SQL Injection, XSS, CSRF 등
- - **System Hacking** - Buffer Overflow, ROP 등
  - - **Reversing** - 바이너리 분석, 디버깅
    - - **CTF Writeup** - 대회 문제 풀이
      - - **Tool & Tip** - 유용한 도구와 팁
       
        - ### 마크다운 테스트
       
        - 코드 블록도 잘 동작합니다:
       
        - ```python
          # Hello Hack Lab!
          import socket

          def scan_port(host, port):
              s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
              s.settimeout(1)
              result = s.connect_ex((host, port))
              s.close()
              return result == 0

          print("Port scan complete!")
          ```

          > 이 블로그의 모든 내용은 **교육 목적**으로 작성되었습니다.
          > > 실제 시스템에 대한 무단 침입은 불법입니다.
          > >
          > > 인라인 코드도 테스트: `nmap -sV -sC target.com`
          > >
          > > ### 앞으로의 계획
          > >
          > > 꾸준히 공부하고, 배운 것들을 정리해서 올리겠습니다. 같이 공부하실 분들은 댓글 남겨주세요!
