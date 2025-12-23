

- **바이트 스트림**: InputStream, OutputStream (바이너리)
- **문자 스트림**: Reader, Writer (텍스트)

---

## 바이트 스트림

### FileInputStream / FileOutputStream
```java
// 읽기
try (FileInputStream fis = new FileInputStream("input.txt")) {
    int data;
    while ((data = fis.read()) != -1) {
        System.out.print((char) data);
    }
}

// 쓰기
try (FileOutputStream fos = new FileOutputStream("output.txt")) {
    fos.write("Hello".getBytes());
}

// 이어쓰기
try (FileOutputStream fos = new FileOutputStream("output.txt", true)) {
    fos.write("추가".getBytes());
}
```

### 버퍼 사용 (성능 향상)
```java
try (FileInputStream fis = new FileInputStream("input.txt")) {
    byte[] buffer = new byte[1024];
    int len;
    while ((len = fis.read(buffer)) != -1) {
        // buffer[0] ~ buffer[len-1] 처리
    }
}
```

### BufferedInputStream / BufferedOutputStream
```java
try (BufferedInputStream bis = new BufferedInputStream(new FileInputStream("input.txt"));
     BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream("output.txt"))) {
    
    byte[] buffer = new byte[1024];
    int len;
    while ((len = bis.read(buffer)) != -1) {
        bos.write(buffer, 0, len);
    }
}
```

---

## 문자 스트림

### FileReader / FileWriter
```java
// 읽기
try (FileReader fr = new FileReader("input.txt")) {
    int data;
    while ((data = fr.read()) != -1) {
        System.out.print((char) data);
    }
}

// 쓰기
try (FileWriter fw = new FileWriter("output.txt")) {
    fw.write("안녕하세요");
}
```

### BufferedReader / BufferedWriter
```java
// 한 줄씩 읽기
try (BufferedReader br = new BufferedReader(new FileReader("input.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}

// 쓰기
try (BufferedWriter bw = new BufferedWriter(new FileWriter("output.txt"))) {
    bw.write("첫 번째 줄");
    bw.newLine();  // 줄바꿈
    bw.write("두 번째 줄");
}
```

---

## Files 클래스 (Java 7+)

### 간편한 파일 읽기/쓰기
```java
Path path = Path.of("file.txt");

// 전체 읽기
String content = Files.readString(path);
List<String> lines = Files.readAllLines(path);
byte[] bytes = Files.readAllBytes(path);

// 전체 쓰기
Files.writeString(path, "내용");
Files.write(path, lines);
Files.write(path, bytes);

// 이어쓰기
Files.writeString(path, "추가", StandardOpenOption.APPEND);
```

### 파일/디렉토리 작업
```java
// 생성
Files.createFile(Path.of("new.txt"));
Files.createDirectory(Path.of("newDir"));
Files.createDirectories(Path.of("a/b/c"));

// 복사/이동/삭제
Files.copy(source, target);
Files.move(source, target);
Files.delete(path);
Files.deleteIfExists(path);

// 확인
Files.exists(path);
Files.isDirectory(path);
Files.isRegularFile(path);
Files.size(path);
```

---

## try-with-resources
```java
// 자동 close()
try (FileReader fr = new FileReader("file.txt");
     BufferedReader br = new BufferedReader(fr)) {
    // 사용
} // 자동으로 close() 호출

// Java 9+: 외부 선언 가능
FileReader fr = new FileReader("file.txt");
try (fr) {
    // 사용
}
```

---

## 실전 예제

### 파일 복사
```java
public static void copyFile(String src, String dest) throws IOException {
    try (InputStream is = new FileInputStream(src);
         OutputStream os = new FileOutputStream(dest)) {
        byte[] buffer = new byte[8192];
        int len;
        while ((len = is.read(buffer)) != -1) {
            os.write(buffer, 0, len);
        }
    }
}

// 간단하게
Files.copy(Path.of("src.txt"), Path.of("dest.txt"));
```

### 텍스트 파일 처리
```java
// 줄 번호 붙이기
try (BufferedReader br = new BufferedReader(new FileReader("input.txt"));
     BufferedWriter bw = new BufferedWriter(new FileWriter("output.txt"))) {
    
    String line;
    int num = 1;
    while ((line = br.readLine()) != null) {
        bw.write(num++ + ": " + line);
        bw.newLine();
    }
}
```

---

> [!tip] 핵심 정리
> 
> - **바이트**: InputStream/OutputStream (바이너리)
> - **문자**: Reader/Writer (텍스트)
> - **버퍼**: Buffered~ (성능 향상)
> - **Files**: 간편한 유틸리티
> - **try-with-resources**: 자동 close()

---

#Java #IO #파일입출력 #Stream