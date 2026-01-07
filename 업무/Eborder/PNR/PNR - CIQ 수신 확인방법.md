pushPnrManager_v3.log (항공사에서 직접 CIQ로 보낸 메세지)
pnrCollectingScheduler_v3_kor (FlightCollectingScheduler로 항공사로 가져온 후 항공사로 가져와 CIQ로 넣는 메세지)


/pnr/log/pushPnrManager_v3_kor
```
I, [2025-12-15T11:08:30.384417 #2049641]  INFO -- : Parser process succeeded (in-2512150207242329.RCV)
I, [2025-12-15T11:08:38.747029 #2049641]  INFO -- : Sender process started (ID: 1340232, 7C1153/20251215/PUS/NRT/2025-12-15 11:05/2025-12-15 13:10)
I, [2025-12-15T11:08:38.750664 #2049641]  INFO -- : Sender process succeeded (ID: 1340232, 7C1153/20251215/PUS/NRT/2025-12-15 11:05/2025-12-15 13:10, /home/eborder/pnr/data.kor/push/7C/inbound/20251215/in-2512150207242329.RCV)
```
/pnr/log/pnrCollectingScheduler_v3_kor
```
I, [2025-12-15T11:00:33.409472 #2675287]  INFO -- : Parser process started (ScheduleID: 157469047, OZ9723/20251216/KIX/PUS)
I, [2025-12-15T11:00:33.621852 #2675287]  INFO -- : Parser process succeeded (ScheduleID: 157469047, OZ9723/20251216/KIX/PUS)
I, [2025-12-15T11:00:35.920324 #2675287]  INFO -- : Sender started (ScheduleID: 157469047, OZ9723/20251216/KIX/PUS)
I, [2025-12-15T11:00:35.922687 #2675287]  INFO -- : Sender process succeeded (ScheduleID: 157469047, OZ9723/20251216/KIX/PUS)
```

위 경로 
succeeded log 확인 후 

항공편명 7C1153 출발일시 2025-12-15 11:05 도착일시 2025-12-15 13:10
![[Pasted image 20251215114031.png]]
항공편명 OZ9723 출발일시 20251216
![[Pasted image 20251215114222.png]]


확인 되면 완료