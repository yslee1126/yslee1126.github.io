---
title: Jenkins Plugin Update 표시 안되는 현상 수정  
date: 2025-08-13 17:00:00 +0900
categories: [infra]
math: true
mermaid: true
---

### 개요
- jenkins 관리 > 플러그인 > 업데이트 목록에서 업데이트 되어야할 플러그인이 목록에 뜨지 않아서 업데이트가 안될때 해결방법

### 설정 
- 젠킨스 설치 폴더 > update 에서 default.json 을 다른 곳에 옮겨놓고 삭제 
- 젠킨스 재시작 /restart 
- 젠킨스 관리 > 플러그인 > 고급설정에 보면 업데이트 서버 주소가 있는데 이 주소를 사용해서 default.json 을 다시 내려받게 된다 
- default.json 이 갱신되면 플러그인 업데이트 목록이 새롭게 갱신되어 필요한 플러그인 업데이트를 진행 할 수 있다 

### 회고 
- 오랜만에 젠킨스 서버를 들어가니 기억이 안난다 
- 완전 관리형 CI/CD 툴을 사용했으면 좋겠다  
