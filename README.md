# Home Server

Docker Compose 기반으로 `SecondaryBook`과 `TripToN`을 한 서버에서 실행하고, Nginx 컨테이너가 외부 요청을 각 앱 컨테이너로 프록시합니다.

## 전체 연결 구조

```text
외부 사용자
  -> http://공인IP:8084 -> nginx -> secondarybook:8080
  -> http://공인IP:8085 -> nginx -> tripton:8085
```

실제 요청 흐름은 다음과 같습니다.

```text
8084 요청 -> home-nginx 컨테이너 -> secondarybook 컨테이너의 Tomcat 8080
8085 요청 -> home-nginx 컨테이너 -> tripton 컨테이너의 Spring Boot 8085
```

## Docker Compose 구조

```text
home-server/
  docker-compose.yml
  .env
  nginx/
    secondarybook.conf
    tripton.conf
```

Nginx 컨테이너가 외부 포트를 받습니다.

```yaml
nginx:
  ports:
    - "8084:8084"
    - "8085:8085"
  volumes:
    - ./nginx:/etc/nginx/conf.d:ro
```

앱 컨테이너들은 외부에 직접 포트를 열지 않고, Docker 내부 네트워크에서만 노출합니다.

```yaml
tripton:
  expose:
    - "8085"

secondarybook:
  expose:
    - "8080"
```

## Nginx 라우팅

`nginx/secondarybook.conf`

```nginx
server {
    listen 8084;

    location / {
        proxy_pass http://secondarybook:8080;
    }
}
```

`nginx/tripton.conf`

```nginx
server {
    listen 8085;

    location / {
        proxy_pass http://tripton:8085;
    }
}
```

## SecondaryBook

SecondaryBook은 별도 Tomcat 설치가 필요 없습니다. Dockerfile 안에서 Tomcat 이미지를 사용합니다.

```text
Maven으로 WAR 빌드
-> Tomcat 9 이미지에 ROOT.war 배포
-> 컨테이너 시작 시 catalina.sh run
-> 컨테이너 내부 8080으로 서비스
```

관련 컨테이너:

```text
secondarybook
secondarybook-mysql
secondarybook-redis
```

DB 연결:

```text
secondarybook -> secondarybook-mysql:3306
```

Redis 연결:

```text
secondarybook -> secondarybook-redis:6379
```

## TripToN

TripToN은 `tripton:local` 이미지를 사용하고, 내부 포트 `8085`로 실행됩니다.

관련 컨테이너:

```text
tripton
tripton-mariadb
```

DB 연결:

```text
tripton -> mariadb:3306
```

## Docker 실행 방법

홈서버 폴더로 이동한 뒤 실행합니다.

```bash
cd /Users/aaa/Desktop/home-server
docker compose up -d --build
```

상태 확인:

```bash
docker compose ps
```

중지:

```bash
cd /Users/aaa/Desktop/home-server
docker compose down
```

로그 보기:

```bash
cd /Users/aaa/Desktop/home-server
docker compose logs -f
```

로컬 접속 확인:

```text
http://localhost:8084 -> SecondaryBook
http://localhost:8085 -> TripToN
```

## 외부 접속

홈서버 Mac의 내부 IP를 확인합니다.

```bash
ifconfig | grep "inet "
```

현재 홈서버 내부 IP는 다음 값으로 사용합니다.

```text
192.168.0.15
```

공유기 관리자 페이지에 접속합니다.

```text
http://192.168.0.1
```

먼저 `DHCP 고정 할당`, `주소 예약`, `IP/MAC 바인딩` 같은 메뉴에서 홈서버 Mac의 내부 IP를 고정합니다.

```text
홈서버 IP: 192.168.0.15
```

그 다음 `포트포워딩`, `NAT`, `가상 서버` 같은 메뉴에서 규칙 2개를 추가합니다.

```text
이름: SecondaryBook
프로토콜: TCP
외부 포트: 8084
내부 IP: 192.168.0.15
내부 포트: 8084
```

```text
이름: TripToN
프로토콜: TCP
외부 포트: 8085
내부 IP: 192.168.0.15
내부 포트: 8085
```

공유기 설정 후 휴대폰 와이파이를 끄고 LTE/5G에서 접속을 테스트합니다.

공인 IP는 네이버에 `내 아이피`를 검색해서 확인합니다.

```text
http://공인IP:8084 -> SecondaryBook
http://공인IP:8085 -> TripToN
```

## 주의

`3307`, `3308` DB 포트는 외부에 포트포워딩하지 않는 것이 좋습니다. 웹 접속용으로는 `8084`, `8085`만 열면 됩니다.

`.env`에는 API 키, DB 비밀번호, OAuth 키가 들어가므로 Git에 올리면 안 됩니다. 이 저장소에는 실제 값 대신 `.env.example`만 포함합니다.
