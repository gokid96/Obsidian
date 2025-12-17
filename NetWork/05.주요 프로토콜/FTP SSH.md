## FTP (File Transfer Protocol)

- **포트**: 21 (제어), 20 (데이터)
- **특징**: 파일 전송 전용, 평문 전송

### Active vs Passive 모드

```
Active Mode:
Client:5000 ──▶ Server:21  (제어 연결)
Client:5001 ◀── Server:20  (서버가 데이터 연결 시작)
→ 클라이언트 방화벽 문제 발생 가능

Passive Mode:
Client:5000 ──▶ Server:21  (제어 연결)
Client:5001 ──▶ Server:3000 (클라이언트가 데이터 연결 시작)
→ 방화벽 친화적
```

### SFTP vs FTPS

|구분|SFTP|FTPS|
|---|---|---|
|기반|SSH (포트 22)|FTP + TLS|
|포트|22|990 또는 21|
|방화벽|단일 포트로 간편|다중 포트 필요|

---

## SSH (Secure Shell)

- **포트**: 22
- **용도**: 원격 접속, 파일 전송, 터널링

### 인증 방식

1. **비밀번호 인증**: 간단하지만 보안 취약
2. **공개키 인증**: 권장 방식

### 공개키 인증 설정

```bash
# 키 생성
ssh-keygen -t ed25519 -C "comment"

# 공개키 서버에 복사
ssh-copy-id user@server
# 또는 수동으로 ~/.ssh/authorized_keys에 추가

# 접속
ssh user@server
```

### SSH 터널링

```bash
# Local Port Forwarding
# 로컬:3306 → 원격 서버 경유 → DB:3306
ssh -L 3306:db-server:3306 user@jump-server

# Remote Port Forwarding
# 원격:8080 → 로컬:3000
ssh -R 8080:localhost:3000 user@remote-server
```