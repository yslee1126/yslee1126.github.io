---
title: Nginx 에서의 Rate Limit 설정 
date: 2025-07-18 17:00:00 +0900
categories: [infra]
math: true
mermaid: true
---

### 개요 
- DDOS 공격으로 IDC 의 WAF 가 역할을 못하고 다운되는 사고가 발생했다  
  - 제품의 문제인지 IDC 환경의 문제인지 불명확한 상황
  - 수많은 트래픽이 WAF를 바이패스 하여 서비스에 도달하는 참사가 발생 
  - WAS 는 난리가남 TCP 65535 개 전부 소진 등등 
  - 그 와중에 제일 멀정했던 부분은 nginx 로 확인됨 
- 다음 번 참사를 막기위해 nginx 에 Rate Limit 기능을 구현 

### 설정 
- nginx 
  - 특정 IP 에 대하여 초당 몇개의 요청을 허용할 것인가
  - nginx 는 허용개수를 참고하여 bucket 을 늘려나간다 예를 들어 초당 10개 허용한다면 0.1초에 버켓이 하나 생기는 형태 
  - 요청이 몰려서 허용 버켓을 초과하는 만큼 503 에러를 리턴하므로 503.html 준비되어 있으면 된다 
  - 단, 처음에는 burst 만큼 요청이 초과되어도 받아준다


```
# zone 설정은 nginx.conf 에 글로벌하게 
limit_req_zone $binary_remote_addr zone=req_limit_ip:메로리m rate=허용개수r/s;

# 서비스별로 zone 적용 conf.d/서비스별.conf 
limit_req zone=req_limit_ip burst=버스트값설정 nodelay;
```

- fail2ban 
  - jail.local 에 차단 정책을 적용한다 
  - findtime 는 의심스러운 활동을 감지할 시간창(초단위)
  - findtime 내에 단일 IP 에 대하여 허용할 이벤트 개수를 maxretry 에 설정 
  - 허용한 이벤트보다 초과 활동이 감지되면 bantime 만큼 차단 


```
backend = auto
enabled = true
filter = nginx-ddos
logpath = nginx 로그 경로/xxxx_access.log
findtime = 10
maxretry = 300
bantime = 60
action = route-blackhole
```

### 회고 
- IDC 레거시 시스템을 MSA 로 전환 할 수록 게이트웨이의 필요성을 느낀다
  - 서비스별로 서버를 찾아서 설정을 바꾸는게 보통 작업이 아니다
  - API 레벨에서는 이미 Rate Limit 이 적용된 서비스가 있는데 이번에 설정한 정책과 공존해야 하는지 논의가 필요하다 
- GCP or AWS 의 WAF 였다면 이정도 DDOS 에 무력화 되었을까 궁금해진다   
- 정책의 수치(임계값)을 정하려고하는데 네트워크팀에서 지난 공격을 포함한 트래픽 통계 데이터를 주셔야 할 것 같다 
  - 아직은 회신이 없다