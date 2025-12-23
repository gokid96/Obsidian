## NIO란?

- **New I/O** (Java 1.4)
- 기존 IO보다 **고성능**
- 논블로킹 지원

---

## IO vs NIO

|구분|IO|NIO|
|---|---|---|
|방향|단방향|양방향 (Channel)|
|처리 단위|Stream (바이트)|Buffer|
|블로킹|블로킹|논블로킹 가능|
|핵심|Stream|Buffer, Channel, Selector|

---

## Buffer

### 주요 Buffer 타입
```java
ByteBuffer    // 가장 많이 사용
CharBuffer
IntBuffer
LongBuffer
DoubleBuffer
// ...
```

### Buffer 생성
```java
// 힙 버퍼
ByteBuffer heapBuffer = ByteBuffer.allocate(1024);

// 다이렉트 버퍼 (OS 메모리, 고성능)
ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024);

// 기존 배열로 생성
byte[] arr = new byte[1024];
ByteBuffer wrapBuffer = ByteBuffer.wrap(arr);
```

### Buffer 속성
```
0 <= mark <= position <= limit <= capacity
```

|속성|설명|
|---|---|
|capacity|전체 크기|
|position|현재 위치|
|limit|읽기/쓰기 한계|
|mark|마킹 위치|

### Buffer 사용
```java
ByteBuffer buffer = ByteBuffer.allocate(10);

// 쓰기
buffer.put((byte) 'H');
buffer.put((byte) 'i');
// position: 2, limit: 10

// 읽기 모드 전환
buffer.flip();
// position: 0, limit: 2

// 읽기
while (buffer.hasRemaining()) {
    System.out.print((char) buffer.get());
}

// 초기화
buffer.clear();     // position=0, limit=capacity
buffer.rewind();    // position=0 (limit 유지)
buffer.compact();   // 읽지 않은 데이터 앞으로 이동
```

---

## Channel

### FileChannel
```java
// 읽기
try (FileChannel channel = FileChannel.open(Path.of("file.txt"), 
        StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    while (channel.read(buffer) != -1) {
        buffer.flip();
        // 처리
        buffer.clear();
    }
}

// 쓰기
try (FileChannel channel = FileChannel.open(Path.of("file.txt"),
        StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {
    ByteBuffer buffer = ByteBuffer.wrap("Hello NIO".getBytes());
    channel.write(buffer);
}
```

### 파일 복사
```java
try (FileChannel src = FileChannel.open(Path.of("src.txt"), StandardOpenOption.READ);
     FileChannel dest = FileChannel.open(Path.of("dest.txt"), 
             StandardOpenOption.WRITE, StandardOpenOption.CREATE)) {
    src.transferTo(0, src.size(), dest);
}
```

---

## Path & Files

### Path
```java
Path path = Path.of("dir", "subdir", "file.txt");
Path path = Paths.get("dir/subdir/file.txt");

path.getFileName();     // file.txt
path.getParent();       // dir/subdir
path.toAbsolutePath();  // 절대 경로
path.resolve("other");  // 경로 합치기
path.normalize();       // . .. 정리
```

### Files 유틸리티
```java
// 스트림으로 읽기 (대용량)
try (Stream<String> lines = Files.lines(Path.of("file.txt"))) {
    lines.filter(line -> line.contains("error"))
         .forEach(System.out::println);
}

// 디렉토리 탐색
try (Stream<Path> paths = Files.walk(Path.of("dir"))) {
    paths.filter(Files::isRegularFile)
         .forEach(System.out::println);
}

// 디렉토리 내용
try (DirectoryStream<Path> stream = Files.newDirectoryStream(Path.of("dir"), "*.txt")) {
    for (Path entry : stream) {
        System.out.println(entry);
    }
}
```

---

## 비동기 채널 (NIO.2)

### AsynchronousFileChannel
```java
AsynchronousFileChannel channel = AsynchronousFileChannel.open(
    Path.of("file.txt"), StandardOpenOption.READ);

ByteBuffer buffer = ByteBuffer.allocate(1024);

// Future 방식
Future<Integer> result = channel.read(buffer, 0);
while (!result.isDone()) {
    // 다른 작업
}
int bytesRead = result.get();

// Callback 방식
channel.read(buffer, 0, null, new CompletionHandler<Integer, Void>() {
    @Override
    public void completed(Integer result, Void attachment) {
        System.out.println("읽기 완료: " + result + " bytes");
    }
    
    @Override
    public void failed(Throwable exc, Void attachment) {
        exc.printStackTrace();
    }
});
```

---

## WatchService

### 파일 변경 감지
```java
WatchService watcher = FileSystems.getDefault().newWatchService();

Path dir = Path.of("watchDir");
dir.register(watcher, 
    StandardWatchEventKinds.ENTRY_CREATE,
    StandardWatchEventKinds.ENTRY_DELETE,
    StandardWatchEventKinds.ENTRY_MODIFY);

while (true) {
    WatchKey key = watcher.take();  // 블로킹
    
    for (WatchEvent<?> event : key.pollEvents()) {
        WatchEvent.Kind<?> kind = event.kind();
        Path filename = (Path) event.context();
        
        System.out.println(kind + ": " + filename);
    }
    
    key.reset();  // 다음 이벤트 받기 위해
}
```

---

## 실전 예제

### 대용량 파일 처리
```java
// Memory Mapped File (초고속)
try (FileChannel channel = FileChannel.open(Path.of("large.dat"), 
        StandardOpenOption.READ)) {
    MappedByteBuffer buffer = channel.map(
        FileChannel.MapMode.READ_ONLY, 0, channel.size());
    
    while (buffer.hasRemaining()) {
        byte b = buffer.get();
        // 처리
    }
}
```

### 파일 속성
```java
Path path = Path.of("file.txt");

// 기본 속성
BasicFileAttributes attrs = Files.readAttributes(path, BasicFileAttributes.class);
attrs.size();
attrs.creationTime();
attrs.lastModifiedTime();
attrs.isDirectory();

// 수정
Files.setLastModifiedTime(path, FileTime.fromMillis(System.currentTimeMillis()));
```

---

> [!tip] 핵심 정리
> 
> - **Buffer**: 데이터 담는 컨테이너 (flip, clear, rewind)
> - **Channel**: 양방향 통로
> - **Path**: 경로 표현
> - **Files**: 파일 유틸리티
> - **WatchService**: 파일 변경 감지

---

#Java #NIO #Buffer #Channel #Path