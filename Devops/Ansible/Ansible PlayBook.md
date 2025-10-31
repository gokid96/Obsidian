## 1. 조건문 (when) - 디스크 사용량 체크

```yaml
---
- name: Conditional Tasks                    # 플레이북 이름
  hosts: devservers                          # 실행할 서버 그룹
  tasks:
    - name: Check disk usage                 # 디스크 사용량 확인 작업
      shell: df -h / | tail -1 | awk '{print $5}' | sed 's/%//'  # 디스크 사용률을 숫자만 추출
      register: disk_usage                   # 결과를 disk_usage 변수에 저장
    
    - name: Warning if disk usage > 80%      # 디스크 사용량이 높을 때 경고
      debug:
        msg: "⚠️ Disk usage is {{ disk_usage.stdout }}% - Need cleanup!"  # 경고 메시지
      when: disk_usage.stdout|int > 80       # 조건: 디스크 사용량이 80%를 넘으면 실행
    
    - name: OK if disk usage < 80%           # 디스크 사용량이 정상일 때
      debug:
        msg: "✅ Disk usage is {{ disk_usage.stdout }}% - Normal"  # 정상 메시지
      when: disk_usage.stdout|int <= 80      # 조건: 디스크 사용량이 80% 이하면 실행
```

## 2. 반복문 (loop) - 여러 패키지 작업을 한번에

```yaml
---
- name: Loop Tasks                          # 반복 작업 예제
  hosts: devservers                         # 실행할 서버 그룹
  tasks:
    - name: Create multiple directories     # 여러 디렉토리를 한번에 생성
      file:
        path: "/tmp/{{ item }}"             # item은 반복되는 각각의 값
        state: directory                    # 디렉토리로 생성
      loop:                                 # 반복할 목록 시작
        - dir1                              # 첫 번째 반복: /tmp/dir1
        - dir2                              # 두 번째 반복: /tmp/dir2
        - dir3                              # 세 번째 반복: /tmp/dir3
    
    - name: Install multiple packages       # 여러 패키지를 한번에 설치
      apt:                                  # Ubuntu/Debian 패키지 관리자
        name: "{{ item }}"                  # 반복되는 패키지 이름
        state: present                      # 설치된 상태로 만듦
      loop:                                 # 설치할 패키지 목록
        - htop                              # 시스템 모니터링 도구
        - curl                              # HTTP 클라이언트
        - wget                              # 파일 다운로드 도구
      become: yes                           # sudo 권한으로 실행 (패키지 설치는 root 권한 필요)
```

## 3. 변수 사용하기 (vars) - 재사용 가능한 값들

```yaml
---
- name: Variables Example                   # 변수 사용 예제
  hosts: devservers                         # 실행할 서버 그룹
  vars:                                     # 변수 정의 섹션
    app_name: myapp                         # 애플리케이션 이름 변수
    app_version: "1.0.0"                    # 애플리케이션 버전 변수
    app_port: 8080                          # 애플리케이션 포트 변수
  tasks:
    - name: Create app directory            # 애플리케이션 디렉토리 생성
      file:
        path: "/opt/{{ app_name }}"         # 변수 사용: /opt/myapp
        state: directory                    # 디렉토리로 생성
    
    - name: Create config file              # 설정 파일 생성
      copy:
        content: |                          # 여러 줄 내용 시작
          APP_NAME={{ app_name }}           # 변수를 파일 내용에 포함
          APP_VERSION={{ app_version }}     # 버전 정보
          APP_PORT={{ app_port }}           # 포트 정보
        dest: "/opt/{{ app_name }}/config.env"  # 저장 위치에도 변수 사용
```

## 4. 파일 템플릿 (template) - 동적 파일 생성

**먼저 templates 디렉토리 생성:**

```bash
mkdir templates
```

**templates/nginx.conf.j2 파일 생성:**

```nginx
# Nginx 설정 템플릿 파일 (Jinja2 템플릿 엔진 사용)
server {
    listen {{ nginx_port }};                # 변수로 포트 설정
    server_name {{ server_name }};          # 변수로 서버명 설정
    root /var/www/{{ app_name }};           # 변수로 문서 루트 설정
    
    # 로그 파일도 변수 사용 가능
    access_log /var/log/nginx/{{ app_name }}_access.log;
    error_log /var/log/nginx/{{ app_name }}_error.log;
}
```

**플레이북:**

```yaml
---
- name: Template Example                    # 템플릿 사용 예제
  hosts: devservers                         # 실행할 서버 그룹
  vars:                                     # 템플릿에서 사용할 변수들
    nginx_port: 80                          # Nginx 포트
    server_name: example.com                # 서버명
    app_name: myapp                         # 애플리케이션 이름
  tasks:
    - name: Generate nginx config           # Nginx 설정 파일 생성
      template:
        src: nginx.conf.j2                  # 템플릿 파일 (templates/ 디렉토리에서 찾음)
        dest: /tmp/nginx.conf               # 생성될 실제 파일 위치
        # 템플릿의 변수들이 실제 값으로 치환되어 파일이 생성됨
```

## 5. 핸들러 (notify/handlers) - 변경시에만 실행

```yaml
---
- name: Handlers Example                    # 핸들러 사용 예제
  hosts: devservers                         # 실행할 서버 그룹
  tasks:
    - name: Update config file              # 설정 파일 업데이트
      copy:
        content: "New configuration"        # 새로운 설정 내용
        dest: /tmp/app.conf                 # 설정 파일 위치
      notify: restart service               # 파일이 변경되면 핸들러 실행 예약
      # notify는 파일이 실제로 변경되었을 때만 핸들러를 호출함
    
    - name: Another task that might change config  # 설정을 변경할 수 있는 다른 작업
      lineinfile:
        path: /tmp/app.conf                 # 기존 파일에 줄 추가
        line: "debug=true"                  # 추가할 내용
      notify: restart service               # 이 작업도 변경시 같은 핸들러 호출
      # 여러 작업이 같은 핸들러를 notify해도 핸들러는 마지막에 한 번만 실행됨
  
  handlers:                                 # 핸들러 정의 섹션
    - name: restart service                 # 핸들러 이름 (notify에서 사용한 이름과 일치해야 함)
      debug:
        msg: "🔄 Service would be restarted here!"  # 실제로는 systemctl restart 등을 사용
      # 핸들러는 모든 일반 task가 끝난 후에 실행됨
      # 변경사항이 있을 때만 실행됨
```

## 6. 종합 예제 - 모든 기능 종합

```yaml
---
- name: Complete Example                    # 모든 기능을 사용한 종합 예제
  hosts: devservers                         # 실행할 서버 그룹
  vars:                                     # 플레이북 전체에서 사용할 변수들
    packages:                               # 리스트 형태의 변수
      - htop                                # 설치할 패키지 목록
      - curl
    service_name: myapp                     # 서비스 이름 변수
  tasks:
    - name: Check OS                        # 운영체제 확인
      debug:
        msg: "This is Ubuntu"               # Ubuntu일 때 출력할 메시지
      when: ansible_distribution == "Ubuntu"  # 조건: Ubuntu 배포판일 때만 실행
      # ansible_distribution은 Ansible이 자동으로 수집하는 시스템 정보
    
    - name: Install packages               # 패키지 설치 (반복문 + 조건문 조합)
      apt:                                  # Ubuntu 패키지 관리자
        name: "{{ item }}"                  # 반복되는 각 패키지
        state: present                      # 설치된 상태로 만듦
      loop: "{{ packages }}"                # 변수로 정의된 패키지 리스트를 반복
      become: yes                           # sudo 권한으로 실행
      when: ansible_distribution == "Ubuntu"  # Ubuntu일 때만 설치
    
    - name: Create service file            # 시스템 서비스 파일 생성
      copy:
        content: |                          # systemd 서비스 파일 내용
          [Unit]
          Description={{ service_name }}    # 변수를 사용한 서비스 설명
          
          [Service]
          ExecStart=/bin/echo "{{ service_name }} running"  # 변수 사용
        dest: "/tmp/{{ service_name }}.service"  # 변수를 사용한 파일명
      notify: reload systemd                # 파일 변경시 systemd 재로드 예약
  
  handlers:                                 # 핸들러 정의
    - name: reload systemd                  # systemd 재로드 핸들러
      debug:
        msg: "Systemd would be reloaded"    # 실제로는 systemctl daemon-reload 사용
      # 서비스 파일이 변경되었을 때만 실행됨
```

**주요 개념 정리:**

- `{{ variable }}`: 변수 사용 문법
- `when:`: 조건부 실행
- `loop:`: 반복 실행
- `register:`: 결과를 변수에 저장
- `notify:`: 변경시 핸들러 실행 예약
- `become: yes`: sudo 권한으로 실행




app.use('/semaphore', createProxyMiddleware({ target: 'http://localhost:3000', changeOrigin: true, pathRewrite: { '^/semaphore': '' }, ws: true, logLevel: 'debug', // 모든 요청을 프록시하도록 router: function(req) { return 'http://localhost:3000'; } }));