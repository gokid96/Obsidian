#### OSI 7계층

| 계층                     | 역할                     | 예시               |
| ---------------------- | ---------------------- | ---------------- |
| 7. 응용계층 (Application)  | 사용자와 직접 소통, 앱 간 데이터 처리 | HTTP, FTP, SMTP  |
| 6. 표현계층 (Presentation) | 데이터 포맷 변환, 암호화         | JPEG, ASCII, SSL |
| 5. 세션계층 (Session)      | 연결 관리, 세션 유지           | RPC, NetBIOS     |
| 4. 전송계층 (Transport)    | 데이터 전송 신뢰성 제공          | TCP, UDP         |
| 3. 네트워크계층 (Network)    | 경로 설정, IP 주소 기반 전달     | IP, ICMP         |
| 2. 데이터링크계층 (Data Link) | MAC 주소 기반 전달, 오류 검출    | Ethernet, ARP    |
| 1. 물리계층 (Physical)     | 실제 전송 매체, 신호 전달        | 케이블, 전기신호, 광신호   |

#### TCP/IP 4계층

|계층|OSI 매핑|역할|예시|
|---|---|---|---|
|응용계층|OSI 5~7|앱 기반 통신|HTTP, FTP, DNS|
|전송계층|OSI 4|호스트 간 신뢰성 전송|TCP, UDP|
|인터넷계층|OSI 3|IP 기반 경로 설정|IP, ICMP|
|네트워크 인터페이스|OSI 1~2|실제 장치 전달|Ethernet, Wi-Fi|