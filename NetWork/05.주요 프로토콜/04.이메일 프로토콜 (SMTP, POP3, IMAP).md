## SMTP (Simple Mail Transfer Protocol)

- **포트**: 25 (기본), 587 (제출), 465 (SSL)
- **용도**: 메일 발송
- **특징**: Push 방식

## POP3 (Post Office Protocol 3)

- **포트**: 110, 995 (SSL)
- **용도**: 메일 수신 (다운로드)
- **특징**: 서버에서 삭제됨 (기본)

## IMAP (Internet Message Access Protocol)

- **포트**: 143, 993 (SSL)
- **용도**: 메일 수신 (동기화)
- **특징**: 서버에 보관, 여러 기기 동기화

## 이메일 전송 흐름

```
[발신자] → [발신 SMTP 서버] → [수신 SMTP 서버] → [POP3/IMAP] → [수신자]
             ↓                    ↓
        DNS MX 조회          메일박스 저장
```

## POP3 vs IMAP

|구분|POP3|IMAP|
|---|---|---|
|저장 위치|로컬|서버|
|멀티 디바이스|X|O|
|오프라인|O|제한적|
|서버 용량|적게 사용|많이 사용|

## 프로토콜 포트 정리

|프로토콜|포트|전송 계층|
|---|---|---|
|FTP|20, 21|TCP|
|SSH/SFTP|22|TCP|
|SMTP|25, 587|TCP|
|DNS|53|UDP/TCP|
|HTTP|80|TCP|
|POP3|110|TCP|
|IMAP|143|TCP|
|HTTPS|443|TCP|
|SMTPS|465|TCP|
|IMAPS|993|TCP|
|POP3S|995|TCP|