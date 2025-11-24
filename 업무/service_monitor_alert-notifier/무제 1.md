
### alertmanager 서버
```yml
eborder@mercury-dev:~/alertmanager$ cat alertmanager.yml
global:
  smtp_smarthost: 'mail.yht.co.kr:465'
  smtp_from: 'eborder.monitor@yht.co.kr'
  smtp_auth_username: 'eborder.monitor'
  smtp_auth_password: '2B0rder@yuhan'
  smtp_require_tls: false

route:
  receiver: 'default'
  group_by: ['service', 'alertname', 'event_type']
  group_wait: 10s  # 첫 알림 수신 후 그룹화 대기 시간
  group_interval: 10s  # 그룹에 새 알림 추가 시 대기 시간
  repeat_interval: 60m  # 동일 알림 재발송 간격
  routes:
    - match_re:
        event_type: '^(W|E)$'
      receiver: 'warning-error-team'
      continue: false
    - match:
        event_type: 'I'
      receiver: 'info-team'
      continue: false
    - match:
        event_type: 'S'
      receiver: 'su-team'
      continue: false

receivers:
  - name: 'default'
    email_configs:
      - to: 'sungmin360@yht.co.kr'
        headers:
          Subject: '{{ .CommonLabels.service }} - {{ .CommonLabels.event_type }}, {{ .GroupLabels.alertname }}{{ if gt (len .Alerts) 1 }} ({{ len .Alerts }}){{ end }}{{ if .Alerts }}, {{ (index .Alerts 0).Annotations.event_time }}{{ end }}'
        html: |
          {{ range .Alerts }}
          {{ if .Annotations.detail }}
          {{ .Annotations.detail | safeHtml }}
          {{ else }}
          * {{ .Annotations.message }}
          {{ end }}
          {{ end }}
        tls_config:
          insecure_skip_verify: true

  - name: 'warning-error-team'
    email_configs:
      - to: 'sungmin360@yht.co.kr'
        headers:
          Subject: '{{ .CommonLabels.service }} - {{ .CommonLabels.event_type }}, {{ .GroupLabels.alertname }}{{ if gt (len .Alerts) 1 }} ({{ len .Alerts }}){{ end }}{{ if .Alerts }}, {{ (index .Alerts 0).Annotations.event_time }}{{ end }}'
        html: |
          {{ range .Alerts }}
          {{ if .Annotations.detail }}
          {{ .Annotations.detail | safeHtml }}
          {{ else }}
          * {{ .Annotations.message }}
          {{ end }}
          {{ end }}
        tls_config:
          insecure_skip_verify: true

  - name: 'info-team'
    email_configs:
      - to: 'sungmin360@yht.co.kr'
        headers:
          Subject: '{{ .CommonLabels.service }} - {{ .CommonLabels.event_type }}, {{ .GroupLabels.alertname }}{{ if gt (len .Alerts) 1 }} ({{ len .Alerts }}){{ end }}{{ if .Alerts }}, {{ (index .Alerts 0).Annotations.event_time }}{{ end }}'
        html: |
          {{ range .Alerts }}
          {{ if .Annotations.detail }}
          {{ .Annotations.detail | safeHtml }}
          {{ else }}
          * {{ .Annotations.message }}
          {{ end }}
          {{ end }}
        tls_config:
          insecure_skip_verify: true

  - name: 'su-team'
    email_configs:
      - to: 'sungmin360@yht.co.kr'
        headers:
          Subject: '{{ .CommonLabels.service }} - {{ .CommonLabels.event_type }}, {{ .GroupLabels.alertname }}{{ if gt (len .Alerts) 1 }} ({{ len .Alerts }}){{ end }}{{ if .Alerts }}, {{ (index .Alerts 0).Annotations.event_time }}{{ end }}'
        html: |
          {{ range .Alerts }}
          {{ if .Annotations.detail }}
          {{ .Annotations.detail | safeHtml }}
          {{ else }}
          * {{ .Annotations.message }}
          {{ end }}
          {{ end }}
        tls_config:
          insecure_skip_verify: true

```





### 알림을 보낼 원격서버 설정
``` ruby
eborder@kor-pnr-test:~/utility/bin$ cat alertmanager
#!/usr/bin/env ruby
require "yaml"
require "daemons"
require "fileutils"

####################

lMessage = <<-MESSAGE
ERROR: no argument given

Usage: alertmanager <service name> <command> or <service name> <server name> <command>

* where <service name> is one of:
  KorPnr <server name> <command>
  KorApp <server name> <command>
  TwnApp <server name> <command>
  KorPfms <server name> <command>

* where <server name> is one of:
  Prod1
  Prod2
  Test

* where <command> is one of:
  start         start an instance of the application
  stop          stop all instances of the application
  restart       stop all instances and restart them afterwards
  status        show status (PID) of application instances

MESSAGE

case lServiceName = ARGV.shift
  when "KorPnr", "KorApp", "TwnApp", "KorPfms"
    case lServerName = ARGV.shift
      when "Prod1", "Prod2", "Test"
    else
      abort "Invalid Server Name."
    end
else
  abort lMessage
end

lCommand = ARGV.shift

unless (["start", "stop", "restart", "status"].include? lCommand)
  abort "Invalid Command"
end

if lServerName
  lServiceName += lServerName
end

####################

BaseFilePath = File.expand_path "../../", __FILE__
ConfigFilePath = File.join BaseFilePath, "config"

LogFilePath = File.join BaseFilePath, "log", "alertmanager"
FileUtils.mkdir_p LogFilePath unless File.directory? LogFilePath

####################
require(File.join(BaseFilePath, "lib", "alertmanagerMonitor"))

options = {
  :app_name   => "alertmanager_#{lServiceName}",
  :ARGV       => [lCommand],
  :dir_mode   => :normal,
  :dir        => File.join(BaseFilePath, 'pid'),
  :multiple   => false,
  :log_output => true,
  :log_dir    => File.join(BaseFilePath, 'bin'),
  :monitor    => true
}

Daemons.run_proc "alertmanager_#{lServiceName}", options do
  puts Time.now

  lConfigData = YAML.load_file File.join(BaseFilePath, "config/alertmanager_#{lServiceName}.conf")

  EBorder.logger = Logger.new File.join(LogFilePath, "alertmanager.log"), "daily"

  lAlertmanagerMonitor = EBorder::AlertmanagerMonitor.new lServiceName, lConfigData

  Signal.trap "TERM" do
    lThread = Thread.start do
      lAlertmanagerMonitor.stop
    end

    lThread.join

    exit
  end

  lAlertmanagerMonitor.start

  sleep
end
```

``` bash
eborder@kor-pnr-test:~/utility/bin$ cat ../config/alertmanager_KorPnrTest.conf
---
ServiceName: KOR-PNR-TEST

# Alertmanager 설정
Alertmanager:
  URL: http://192.168.14.31:9093

# 모니터링 모듈 설정
Modules:
- ModuleFileName: (#ModuleFilePath)/commonConnectionCheck.rb
  ExecuteTime: 00:00-23:55
  Interval: 300
  Parameter:
    SendingServerList:
    - ServerName: USTD-DEV1
      HostName: 192.168.14.31
      PortNumber: 1112

- ModuleFileName: (#ModuleFilePath)/commonDaemonCheck.rb
  Interval: 60
  Parameter:
    ExcludePidCheck:
    - "edifactParser_v2"
    DaemonInfos:
    - DaemonName: xaAMR
      SpecificDaemonNames:
      - "com.arinc.amr.control.nix.Main"
    - DaemonName: httConnector
      HostNames: 1E,BX
    - DaemonName: sabreConnector_1S
    - DaemonName: flightCollectingScheduler
      MonitorStatus: Y
      SpecificDaemonNames:
      - flightCollectingScheduler_v3_kor
    - DaemonName: pnrCollectingScheduler
      MonitorStatus: Y
      SpecificDaemonNames:
      - pnrCollectingScheduler_v3_kor
    - DaemonName: pushPnrManager_v3_kor
      MonitorStatus: Y
    - DaemonName: edifactMessageParser
      MonitorStatus: Y
    - DaemonName: archiveManager
      MonitorStatus: Y
      SpecificDaemonNames: "archiveManager_PnrTest"
    - DaemonName: mysqld
      SpecificDaemonNames: "/usr/sbin/mysqld"
    - DaemonName: serviceMonitor
      MonitorStatus: Y
      SpecificDaemonNames: "serviceMonitor_KorPnrTest"
    - DaemonName: alertmanager
      MonitorStatus: Y
      SpecificDaemonNames: "alertmanager_KorPnrTest"
  



- ModuleFileName: (#ModuleFilePath)/commonDailyCheck.rb
  ExecuteTime: 09:00,16:30
  Parameter:
    TimeZone: KST

- ModuleFileName: (#ModuleFilePath)/commonDiskCheck.rb
  Interval: 60
  Parameter:
    FilesystemList:
    - Filesystem: /dev/sda3
      DiskSizeUse: 10
      DiskBackup: Y
    - Filesystem: udev
      DiskSizeUse: 5
    - Filesystem: tmpfs
      DiskSizeUse: 15
    - Filesystem: /dev/sda1
      DiskSizeUse: 5
    - Filesystem: /dev/loop0
      DiskSizeUse: 100
    - Filesystem: /dev/loop1
      DiskSizeUse: 100
    - Filesystem: /dev/loop2
      DiskSizeUse: 100

- ModuleFileName: (#ModuleFilePath)/commonInodeCheck.rb
  Interval: 60
  Parameter:
    FilesystemList:
    - Filesystem: /dev/sda3
      INodesUse: 5
      DiskBackup: Y
    - Filesystem: udev
      INodesUse: 5
    - Filesystem: tmpfs
      INodesUse: 5
    - Filesystem: /dev/sda1
      INodesUse: 5
    - Filesystem: /dev/loop0
      INodesUse: 100
    - Filesystem: /dev/loop1
      INodesUse: 100
    - Filesystem: /dev/loop2
      INodesUse: 100


- ModuleFileName: (#ModuleFilePath)/commonRoutingTableCheck.rb
  Interval: 60
  Parameter:
    RoutingTableList:
    - Destination: 0.0.0.0
      Gateway: 192.168.100.1
      Genmask: 0.0.0.0
      Iface: bond1
    - Destination: 192.168.100.0
      Gateway: 0.0.0.0
      Genmask: 255.255.255.0
      Iface: bond1

- ModuleFileName: (#ModuleFilePath)/commonSessionCheck.rb
  Interval: 60
  Parameter:
    TotalSessionCount: 800
    SessionList:
    - SessionName: xaAMR
      SessionPort: 1617
      SessionCount: 80
      SessionCheck: Over
      Alert: N
    - SessionName: DB
      SessionPort: 3306
      SessionCount: 50
      SessionCheck: Over
      Alert: N

- ModuleFileName: (#ModuleFilePath)/commonNoneProcessFileCheck.rb
  Interval: 60
  Parameter:
    DirectoryList:
    - Directory: /home/eborder/xaAMR/kor/inbound/push
      CreatedTime: 10
    - Directory: /home/eborder/xaAMR/kor/outbound/CIQ
      CreatedTime: 10

- ModuleFileName: (#ModuleFilePath)/pnrPushProcessCheck_v3_kor.rb
  Interval: 300
  Parameter:
    LocalAirportCodeFile: "/home/eborder/pnr/config/localAirportCodes_kor.json"
    DatabaseServer:
      HostName: 127.0.0.1
      PortNumber: 3306
      UserName: eborder
      Password: 2Border@19
      DatabaseName: KOR-PNR
      PollTimeout: 60
    PollingInterval: 10
    ProcessTimeover: 10

- ModuleFileName: (#ModuleFilePath)/pnrLastMessageDateUpdater.rb
  ExecuteTime: 08:00,12:00,16:00
  Parameter:
    ServerName: TEST
    MonitoringDatabaseServer:
      HostName: 192.168.100.151
      PortNumber: 3306
      UserName: eborder
      Password: YhtEborder@6277
      DatabaseName: eborder
      PollTimeout: 60
    PNRDatabaseServer:
      HostName: 127.0.0.1
      PortNumber: 3306
      UserName: eborder
      Password: 2Border@19
      PollTimeout: 60

- ModuleFileName: (#ModuleFilePath)/pnrReport.rb
  ExecuteTime: 08:00,18:00
  Parameter:
    TimeZone: KST
    KorDatabaseServer:
      HostName: 127.0.0.1
      PortNumber: 3306
      UserName: eborder
      Password: 2Border@19
      DatabaseName: KOR-PNR
      PollTimeout: 60
    JpnDatabaseServer:
      HostName: 127.0.0.1
      PortNumber: 3306
      UserName: eborder
      Password: 2Border@19
      DatabaseName: JPN-PNR
      PollTimeout: 60
    MonitorDatabaseServer:
      HostName: 192.168.100.151
      PortNumber: 3306
      UserName: eborder
      Password: YhtEborder@6277
      DatabaseName: eborder
      PollTimeout: 60
    ProcessName:
      U: Job Updater
    ReportInterval:
    ExcludeService:
    - 4V
    - TW
    - JQ
    - VJ
    - ZE
```

``` ruby
eborder@kor-pnr-test:~/utility/lib$ cat alertmanagerMonitor.rb
require "json"
require "logger"
require "concurrent"
require "net/http"
require "uri"

require_relative "eBorder"

module EBorder
  AlertmanagerMonitorError = Class.new RuntimeError

  class AlertmanagerMonitor
    CollectingInterval = 4
    SendingInterval = 4

    def initialize aServiceName, aConfigInfo
      @EventInfos = Concurrent::AtomicReference.new []

      lConfigFilePath = File.join BaseFilePath, "config/alertmanager"
      lModuleFilePath = File.join BaseFilePath, "lib/serviceMonitor"

      @ServiceName = aConfigInfo.fetch("ServiceName")
      @EventType = aConfigInfo["EventType"]

      # 모듈 설정
      @ModuleInfos = aConfigInfo.fetch("Modules").map do |aModuleInfo|
        lModuleInfo = {}

        lModuleFileName = aModuleInfo.fetch "ModuleFileName"
        lModuleFileName.sub!(/\(\#ModuleFilePath\)/i, lModuleFilePath)

        lModuleInfo[:Name] = File.basename lModuleFileName
        lModuleInfo[:Command] = ["ruby", lModuleFileName, @ServiceName]
        lModuleInfo[:Status] = :Pending
        lModuleInfo[:Interval] = aModuleInfo["Interval"]
        lModuleInfo[:Time] = Time.local(*Time.now.to_a.fill(0, 0, 3))

        # ExecuteTime 처리
        lExecuteTimes = []
        if lExecuteTime = aModuleInfo["ExecuteTime"]
          if lExecuteTime.is_a? Numeric
            lTimeset = Time.local(*Time.now.to_a.fill(0, 0, 3)).to_i + lExecuteTime
            lExecuteTime = Time.at(lTimeset).strftime("%R")
          end

          lExecuteTime.split(",").each do |aTime|
            lStartEndTime = aTime.strip.split("-")
            lStartTime = Time.strptime(lStartEndTime[0], "%R")
            lEndTime = lStartEndTime[1] ? Time.strptime(lStartEndTime[1], "%R") : lStartTime

            loop do
              lExecuteTimes << lStartTime.strftime("%R")
              lModuleInfo[:Interval] ? lStartTime += lModuleInfo[:Interval] : break
              break if lStartTime > lEndTime
            end
          end

          lModuleInfo[:ExecuteTimes] = lExecuteTimes

          lParameter = aModuleInfo["Parameter"]
          if lParameter
            lParameter["ExecuteTimes"] = "#{lExecuteTimes.first},#{lExecuteTimes.last}"
          end
        end

        lParameter = aModuleInfo["Parameter"]
        lModuleInfo[:Command] << lParameter.to_json if lParameter

        lModuleInfo
      end

      EBorder.logger.info @ModuleInfos

      @ModuleInfosLock = Concurrent::ReadWriteLock.new

      # Alertmanager 설정
      lAlertmanagerConfig = aConfigInfo.fetch "Alertmanager"
      @AlertmanagerURL = lAlertmanagerConfig.fetch "URL"

      EBorder.logger.info "[AlertmanagerMonitor] Target: #{@AlertmanagerURL}"

      # Health check
      if check_alertmanager_health
        EBorder.logger.info "[AlertmanagerMonitor] Health check: ✅ OK"
      else
        EBorder.logger.warn "[AlertmanagerMonitor] Health check: ⚠️ FAILED"
      end

      @CollectingTaskEvent = Concurrent::Event.new
      @SendingTaskEvent = Concurrent::Event.new

      @Executor = Concurrent::ThreadPoolExecutor.new min_threads: 0, max_threads: 8
    end

    # ========== HTTP 전송 메서드 ==========

  def send_to_alertmanager(event_infos)
      return false if event_infos.empty?
    
      # 이벤트 → Alertmanager 형식 변환
      alerts = event_infos.map do |event|
        alert = {
          labels: {
            alertname: event[:EventName] || "Unknown",
            service: event[:ServiceName] || "Unknown",
            event_type: event[:EventType] || "I",
            instance: event[:EventTime] || Time.now.strftime("%Y-%m-%d %H:%M:%S") # fingerprint 구분용
          },
          annotations: {
            message: event[:EventMessage] || "",
            event_time: event[:EventTime] || Time.now.strftime("%Y-%m-%d %H:%M:%S")
          }
        }
    
        # I 타입 (정기 리포트)은 즉시 만료 → 1번만 발송
        if event[:EventType] =~ /I$/
          alert[:endsAt] = (Time.now + 30).utc.strftime("%Y-%m-%dT%H:%M:%S.000Z")
        end
    
        if event[:EventDetailMessage] && !event[:EventDetailMessage].empty?
          alert[:annotations][:detail] = event[:EventDetailMessage]
        end
    
        alert
      end
      
        # HTTP POST
      uri = URI.parse("#{@AlertmanagerURL}/api/v2/alerts")
    
      http = Net::HTTP.new(uri.host, uri.port)
      http.open_timeout = 3
      http.read_timeout = 5
    
      request = Net::HTTP::Post.new(uri.path, {'Content-Type' => 'application/json'})
      request.body = alerts.to_json
    
      EBorder.logger.info "[Alertmanager] Sending #{alerts.size} alert(s) to #{@AlertmanagerURL}"
    
      response = http.request(request)
    
      if response.code.to_i >= 200 && response.code.to_i < 300
        EBorder.logger.info "[Alertmanager] Success - Response: #{response.code}"
        true
      else
        EBorder.logger.error "[Alertmanager] Failed - Response: #{response.code}"
        false
      end
    
    rescue Timeout::Error => e
      EBorder.logger.warn "[Alertmanager] Timeout: #{e.message}"
      false
    rescue Errno::ECONNREFUSED => e
      EBorder.logger.warn "[Alertmanager] Connection refused: #{e.message}"
      false
    rescue => e
      EBorder.logger.error "[Alertmanager] Error: #{e.class} - #{e.message}"
      false
    end

    # ========== 모니터링 로직 ==========

    def start
      return if @Started

      @EventInfos.set []

      @CollectingTaskEvent.reset
      @SendingTaskEvent.reset

      # 모듈 실행 Task
      @CollectingTask = Concurrent::Future.new do
        begin
          until @CollectingTaskEvent.wait CollectingInterval
            @ModuleInfosLock.with_write_lock do
              lTime = Time.now

              @ModuleInfos.each do |lModuleInfo|
                next if lModuleInfo[:Status] == :Running

                # ExecuteTime 체크
                if lExecuteTimes = lModuleInfo[:ExecuteTimes]
                  lFindExecuetTime = lExecuteTimes.find do |aExecuteTime|
                    aExecuteTime == lTime.strftime("%R")
                  end

                  if lFindExecuetTime
                    next if lModuleInfo[:Status] == :Holding
                  else
                    lModuleInfo[:Status] = :Pending unless lModuleInfo[:Status] == :Pending
                    next
                  end
                else
                  next unless lModuleInfo[:Time] + lModuleInfo[:Interval] < lTime
                end

                lModuleInfo[:Status] = :Running

                @Executor.post lModuleInfo do |aModuleInfo|
                  begin
                    lResult = IO.popen aModuleInfo[:Command] do |aIO|
                      aIO.read.strip
                    end

                    lEventInfos = JSON.parse lResult, symbolize_names: true

                    @EventInfos.update do |aEventInfos|
                      aEventInfos + lEventInfos
                    end

                  rescue Exception => lException
                    lExceptionInfos = {}
                    lExceptionInfos[:ServiceName] = @ServiceName
                    lExceptionInfos[:EventType] = "TE"
                    lExceptionInfos[:EventName] = "Failed to execute module (#{aModuleInfo[:Name]})"
                    lExceptionInfos[:EventMessage] = "#{lException.message} (#{lException.class})"
                    lExceptionInfos[:EventDetailMessage] = "#{lException.backtrace.join "\n  "}"
                    lExceptionInfos[:EventTime] = Time.now.strftime("%F %T")

                    @EventInfos.update do |aEventInfos|
                      aEventInfos + [lExceptionInfos]
                    end

                    EBorder.logger.error "Failed to execute module. (#{aModuleInfo[:Name]})\n\n#{lResult}\n"
                    EBorder.logger.error "#{lException.message} (#{lException.class})\n\n  #{lException.backtrace.join "\n  "}\n"

                  else
                    EBorder.logger.info "Module executed successfully. (#{aModuleInfo[:Name]})\n\n#{lResult}\n"

                  ensure
                    @ModuleInfosLock.with_write_lock do
                      aModuleInfo[:Status] = lModuleInfo[:ExecuteTimes] ? :Holding : :Pending
                      aModuleInfo[:Time] = Time.now
                    end
                  end
                end
              end
            end
          end

        rescue Exception => lException
          EBorder.logger.error "#{lException.message} (#{lException.class})\n\n  #{lException.backtrace.join "\n  "}\n"
          retry
        end
      end

      # Alertmanager 전송 Task
      @SendingTask = Concurrent::Future.new do
        begin
          until @SendingTaskEvent.wait SendingInterval do
            next unless @EventInfos.value.count > 0

            lEventInfos = @EventInfos.swap []

            begin
              success = send_to_alertmanager(lEventInfos)

              if success
                EBorder.logger.info "[Alertmanager] EventMessage sent successfully. (#{lEventInfos.count} events)"
              else
                # 재시도를 위해 다시 큐에 넣기
                @EventInfos.update do |aEventInfos|
                  lEventInfos + aEventInfos
                end

                EBorder.logger.error "[Alertmanager] Failed to send EventMessage. (#{lEventInfos.count} events)"
              end

            rescue Exception => lException
              @EventInfos.update do |aEventInfos|
                lEventInfos + aEventInfos
              end

              EBorder.logger.error "[Alertmanager] Failed to send EventMessage. (#{lEventInfos.count} events)"
              EBorder.logger.error "#{lException.message} (#{lException.class})\n\n  #{lException.backtrace.join "\n  "}\n"
            end
          end

        rescue Exception => lException
          EBorder.logger.error "#{lException.message} (#{lException.class})\n\n  #{lException.backtrace.join "\n  "}\n"
          retry
        end
      end

      @CollectingTask.execute
      @SendingTask.execute

      @Started = true

      EBorder.logger.info "######## AlertmanagerMonitor started."
    end

    def stop
      return unless @Started

      @CollectingTask.cancel
      @SendingTask.cancel

      @CollectingTaskEvent.set
      @SendingTaskEvent.set

      @CollectingTask.wait
      @SendingTask.wait

      @Started = false

      EBorder.logger.info "######## AlertmanagerMonitor stopped."
    end
  end
end
```
