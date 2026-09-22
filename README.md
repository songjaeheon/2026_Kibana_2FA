# 🛡️ Kibana Auto-Login & Audit Logging PoC

본 프로젝트는 Windows 환경의 Docker Desktop을 활용하여, Nginx 프록시를 통한 사내 인증(SSO 대체용 Basic Auth) 후 **Kibana 자동 로그인 처리** 및 **접속자 Audit(감사) 로깅**을 최소 규모(PoC)로 검증하기 위한 가이드입니다.

## 📌 아키텍처 개요
1. **Nginx Proxy**: 사용자에게 가상의 사내 인증(Basic Auth) 창을 띄웁니다.
2. **Header Injection (Auto-Login)**: 인증을 통과하면, Nginx가 Kibana 조회 전용 공통 계정(`kibana_viewer`)의 크리덴셜을 Authorization 헤더에 강제 주입하여 Kibana 로그인 창을 우회합니다.
3. **X-Opaque-Id (Audit)**: 모든 사용자가 공통 계정으로 접속하더라도 누가 접속했는지 식별하기 위해, Nginx가 인증된 사용자의 ID를 `X-Opaque-Id` 헤더에 담아 전달하고 Elasticsearch가 이를 Audit 로그에 기록합니다.

## 🚀 사전 준비
- Windows 환경 (PowerShell 사용)
- Docker Desktop 설치 및 실행 (메모리 할당 4GB 이상 권장)

## 🛠️ 실행 및 검증 절차

모든 명령어는 `docker-compose.yml`과 `nginx.conf` 파일이 위치한 폴더의 **PowerShell**에서 실행합니다.

### Step 1. 가상의 사내 인증 DB 생성
`employee` (비밀번호: `emp123`) 계정이 담긴 `.htpasswd` 파일을 생성합니다.
```powershell
docker run --rm httpd:alpine htpasswd -bn employee emp123 > .htpasswd

```

### Step 2. 컨테이너 실행

```powershell
docker-compose up -d

```

> **주의:** Elasticsearch 초기화에 1~2분이 소요됩니다.

### Step 3. Kibana 시스템 계정 비밀번호 초기화 (중요)

Elastic 8.x 보안 정책에 따라 Kibana 내부 통신용 계정(`kibana_system`)의 비밀번호를 세팅합니다.

```powershell
docker exec -it elasticsearch curl -s -X POST "http://localhost:9200/_security/user/kibana_system/_password" -H "Content-Type: application/json" -u "elastic:elastic_admin_password" -d '{ \"password\": \"kibana_system_password\" }'

```

*(성공 시 `{}` 빈 객체가 반환됩니다.)*

### Step 4. Kibana 조회 전용 Role 생성

Kibana를 읽기 전용으로 사용할 수 있는 권한(`kibana_readonly`)을 생성합니다. (PowerShell 따옴표 이스케이프 반영)

```powershell
docker exec -it elasticsearch curl -s -X POST "http://localhost:9200/_security/role/kibana_readonly" -H "Content-Type: application/json" -u "elastic:elastic_admin_password" -d '{ \"cluster\": [], \"indices\": [{\"names\": [\"*\"], \"privileges\": [\"read\", \"view_index_metadata\"]}], \"applications\": [{\"application\": \"kibana-.kibana\", \"privileges\": [\"read\"], \"resources\": [\"*\"]}] }'
```

### Step 5. 공통 Viewer User 생성

Nginx가 헤더에 주입할 실제 계정(`kibana_viewer` / `viewer_password`)을 생성하고 권한을 부여합니다.

```powershell
docker exec -it elasticsearch curl -s -X POST "http://localhost:9200/_security/user/kibana_viewer" -H "Content-Type: application/json" -u "elastic:elastic_admin_password" -d '{ \"password\" : \"viewer_password\", \"roles\" : [ \"kibana_readonly\" ], \"full_name\" : \"Kibana Viewer\" }'

```

### Step 6. 웹 브라우저 검증

1. 브라우저에서 `http://localhost:8080` 으로 접속합니다.
2. 인증 팝업이 뜨면 `employee` / `emp123` 을 입력합니다.
3. Elastic 로그인 화면 없이 즉시 **Kibana 메인 화면**으로 진입합니다.

### Step 7. Audit (감사) 로그 추적 확인

Kibana에서 임의의 메뉴를 클릭한 뒤, 아래 명령어로 접속자(employee)가 기록되었는지 확인합니다.

```powershell
# 실시간 Audit 로그 전체 모니터링
docker logs -f elasticsearch | Select-String '"type":"audit"'
# 접속자 식별용 opaque_id 필터링 모니터링
docker logs -f elasticsearch | Select-String "opaque_id"
```

```powershell
docker exec -it elasticsearch tail -f /usr/share/elasticsearch/logs/docker-cluster_audit.json

```

로그 내에 `"user.name":"kibana_viewer"` 와 함께 **`"opaque_id":"employee"`** 가 기록되어 있다면 성공입니다.

## 🧹 자원 정리

테스트 완료 후 생성된 컨테이너 및 볼륨 데이터를 삭제합니다.

```powershell
docker-compose down -v

```

