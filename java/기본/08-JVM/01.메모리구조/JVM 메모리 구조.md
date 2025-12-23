## JVM 메모리 구조
```
┌─────────────────────────────────────────┐
│             Method Area                 │  ← 클래스 정보, static
├─────────────────────────────────────────┤
│                Heap                     │  ← 객체, 배열
├──────────────────┬──────────────────────┤
│      Stack       │    PC Register       │  ← 스레드별
├──────────────────┼──────────────────────┤
│  Native Method   │                      │
│      Stack       │                      │
└──────────────────┴──────────────────────┘
```

---

## Method Area (메서드 영역)

- **클래스 정보** 저장
- 모든 스레드 공유
```java
public class User {
    // 클래스 메타데이터 → Method Area
    private static int count;        // static 변수 → Method Area
    private static final String TYPE = "USER";  // 상수
    
    public void doSomething() { }    // 메서드 정보
}
```

|저장 내용|
|---|
|클래스 메타데이터 (이름, 접근제어자)|
|필드/메서드 정보|
|static 변수|
|상수 풀 (Runtime Constant Pool)|

---

## Heap (힙)

- **객체, 배열** 저장
- 모든 스레드 공유
- **GC 대상**
```java
User user = new User();      // User 객체 → Heap
int[] arr = new int[10];     // 배열 → Heap
String str = new String("hi"); // String 객체 → Heap
```

### 힙 구조
```
┌─────────────────────────────────────┐
│            Young Generation          │
│  ┌───────┬───────────┬───────────┐  │
│  │ Eden  │ Survivor0 │ Survivor1 │  │
│  └───────┴───────────┴───────────┘  │
├─────────────────────────────────────┤
│            Old Generation            │
└─────────────────────────────────────┘
```

|영역|설명|
|---|---|
|Eden|새 객체 생성|
|Survivor|Minor GC 생존 객체|
|Old|오래 살아남은 객체|

---

## Stack (스택)

- **스레드별** 생성
- 메서드 호출 시 **프레임** 생성
```java
public void method1() {
    int x = 10;           // 지역 변수 → Stack
    User user = new User(); // 참조 변수 → Stack (객체는 Heap)
    method2(x);
}

public void method2(int param) {
    int y = 20;
}
```

### 스택 프레임
```
┌─────────────────┐
│ method2 프레임   │  ← param, y
├─────────────────┤
│ method1 프레임   │  ← x, user(참조)
├─────────────────┤
│   main 프레임    │
└─────────────────┘
```

|구성요소|설명|
|---|---|
|지역 변수|메서드 내 변수|
|매개변수|전달받은 인자|
|리턴 주소|복귀할 위치|
|연산 스택|연산 중간값|

---

## PC Register

- **현재 실행 중인 명령어 주소**
- 스레드별 생성

---

## Native Method Stack

- **네이티브 메서드** (C, C++) 실행
- JNI 사용 시

---

## 예제로 이해하기
```java
public class Main {
    public static void main(String[] args) {
        int num = 10;                    // Stack
        String name = "Kim";             // Stack(참조), Heap(객체)
        
        User user = new User("Lee", 25); // Stack(참조), Heap(객체)
        user.printInfo();
    }
}

class User {
    private static int count = 0;        // Method Area
    private String name;                 // Heap (객체 필드)
    private int age;                     // Heap (객체 필드)
    
    public User(String name, int age) {
        this.name = name;
        this.age = age;
        count++;
    }
    
    public void printInfo() {
        String info = name + ", " + age; // Stack
        System.out.println(info);
    }
}
```
```
Method Area: User 클래스 정보, count
Heap: User 객체, String 객체들
Stack: main() → num, name참조, user참조
       printInfo() → info참조
```

---

## 메모리 관련 에러

|에러|원인|
|---|---|
|StackOverflowError|재귀 무한 호출|
|OutOfMemoryError: Heap|객체 너무 많음|
|OutOfMemoryError: Metaspace|클래스 너무 많음|
```java
// StackOverflowError
public void infinite() {
    infinite();  // 무한 재귀
}

// OutOfMemoryError
List<byte[]> list = new ArrayList<>();
while (true) {
    list.add(new byte[1024 * 1024]);  // 계속 할당
}
```

---

## JVM 옵션
```bash
# 힙 크기
-Xms512m      # 초기 힙
-Xmx1024m     # 최대 힙

# 메타스페이스
-XX:MetaspaceSize=128m
-XX:MaxMetaspaceSize=256m

# 스택 크기
-Xss1m        # 스레드 스택

# GC 로그
-verbose:gc
-XX:+PrintGCDetails
```

---

> [!tip] 핵심 정리
> 
> - **Method Area**: 클래스 정보, static (공유)
> - **Heap**: 객체, 배열 (공유, GC 대상)
> - **Stack**: 지역변수, 매개변수 (스레드별)
> - **PC Register**: 실행 명령어 주소
> - 참조 변수는 Stack, 실제 객체는 Heap

---

#Java #JVM #메모리구조 #Heap #Stack