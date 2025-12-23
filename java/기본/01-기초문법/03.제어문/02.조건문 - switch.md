## 개요

- **하나의 값**을 여러 경우와 비교
- if-else if 체인의 대안
- 가독성이 좋고 성능상 이점이 있을 수 있음

---

## 기본 switch문

### 문법

```java
switch (변수) {
    case 값1:
        // 값1일 때 실행
        break;
    case 값2:
        // 값2일 때 실행
        break;
    default:
        // 어떤 case도 해당 안 될 때
        break;
}
```

### 예시

```java
int day = 3;
String dayName;

switch (day) {
    case 1:
        dayName = "월요일";
        break;
    case 2:
        dayName = "화요일";
        break;
    case 3:
        dayName = "수요일";
        break;
    case 4:
        dayName = "목요일";
        break;
    case 5:
        dayName = "금요일";
        break;
    case 6:
        dayName = "토요일";
        break;
    case 7:
        dayName = "일요일";
        break;
    default:
        dayName = "잘못된 입력";
        break;
}

System.out.println(dayName);  // 수요일
```

---

## 사용 가능한 타입

### Java 버전별

|타입|Java 버전|
|---|---|
|`byte`, `short`, `int`, `char`|1.0+|
|`Byte`, `Short`, `Integer`, `Character`|5.0+|
|`enum`|5.0+|
|`String`|7.0+|

### 예시

```java
// int
switch (number) { ... }

// char
switch (grade) {
    case 'A': ...
    case 'B': ...
}

// String (Java 7+)
switch (command) {
    case "start": ...
    case "stop": ...
}

// enum
enum Season { SPRING, SUMMER, FALL, WINTER }
switch (season) {
    case SPRING: ...
    case SUMMER: ...
}
```

---

## break의 중요성

### break가 없으면 fall-through 발생

```java
int num = 2;

switch (num) {
    case 1:
        System.out.println("1");
    case 2:
        System.out.println("2");  // 실행
    case 3:
        System.out.println("3");  // 이것도 실행!
    default:
        System.out.println("default");  // 이것도 실행!
}
// 출력: 2, 3, default
```

### fall-through 의도적 사용

```java
int month = 2;
String season;

switch (month) {
    case 3:
    case 4:
    case 5:
        season = "봄";
        break;
    case 6:
    case 7:
    case 8:
        season = "여름";
        break;
    case 9:
    case 10:
    case 11:
        season = "가을";
        break;
    case 12:
    case 1:
    case 2:
        season = "겨울";
        break;
    default:
        season = "잘못된 월";
}
```

---

## switch 표현식 (Java 12+)

### 화살표 문법 (->)

```java
int day = 3;

String dayName = switch (day) {
    case 1 -> "월요일";
    case 2 -> "화요일";
    case 3 -> "수요일";
    case 4 -> "목요일";
    case 5 -> "금요일";
    case 6 -> "토요일";
    case 7 -> "일요일";
    default -> "잘못된 입력";
};

System.out.println(dayName);  // 수요일
```

### 여러 case 합치기

```java
String season = switch (month) {
    case 3, 4, 5 -> "봄";
    case 6, 7, 8 -> "여름";
    case 9, 10, 11 -> "가을";
    case 12, 1, 2 -> "겨울";
    default -> "잘못된 월";
};
```

### 블록과 yield (Java 13+)

```java
String result = switch (grade) {
    case "A", "B" -> {
        System.out.println("우수");
        yield "합격";  // 값 반환
    }
    case "C", "D" -> {
        System.out.println("보통");
        yield "합격";
    }
    default -> {
        System.out.println("노력 필요");
        yield "불합격";
    }
};
```

---

## if vs switch

### switch가 좋은 경우

```java
// 하나의 변수를 여러 값과 비교
switch (command) {
    case "start" -> startProcess();
    case "stop" -> stopProcess();
    case "pause" -> pauseProcess();
    case "resume" -> resumeProcess();
    default -> showHelp();
}
```

### if가 좋은 경우

```java
// 범위 비교
if (score >= 90) { }
else if (score >= 80) { }

// 복잡한 조건
if (age >= 18 && hasLicense) { }

// 다른 변수들 비교
if (a > b && c < d) { }
```

### 비교 표

|상황|권장|
|---|---|
|단일 값 비교 (여러 개)|switch|
|범위 비교|if|
|복잡한 조건식|if|
|String/enum 분기|switch|

---

## enum과 switch

### enum 정의

```java
enum Grade {
    A, B, C, D, F
}
```

### switch 사용

```java
Grade grade = Grade.B;

switch (grade) {
    case A:
        System.out.println("최우수");
        break;
    case B:
        System.out.println("우수");
        break;
    case C:
        System.out.println("보통");
        break;
    case D:
        System.out.println("미흡");
        break;
    case F:
        System.out.println("낙제");
        break;
}

// Java 12+ 표현식
String result = switch (grade) {
    case A -> "최우수";
    case B -> "우수";
    case C -> "보통";
    case D -> "미흡";
    case F -> "낙제";
};
```

---

## 실전 예제

### 계산기

```java
int a = 10, b = 3;
char op = '+';

int result = switch (op) {
    case '+' -> a + b;
    case '-' -> a - b;
    case '*' -> a * b;
    case '/' -> a / b;
    case '%' -> a % b;
    default -> throw new IllegalArgumentException("Unknown operator: " + op);
};

System.out.println(result);  // 13
```

### 메뉴 처리

```java
String menu = "2";

switch (menu) {
    case "1":
        System.out.println("게임 시작");
        break;
    case "2":
        System.out.println("설정");
        break;
    case "3":
        System.out.println("도움말");
        break;
    case "0":
        System.out.println("종료");
        break;
    default:
        System.out.println("잘못된 입력");
}
```

### 월별 일수

```java
int month = 2;
int year = 2024;

int days = switch (month) {
    case 1, 3, 5, 7, 8, 10, 12 -> 31;
    case 4, 6, 9, 11 -> 30;
    case 2 -> {
        boolean isLeap = (year % 4 == 0 && year % 100 != 0) || year % 400 == 0;
        yield isLeap ? 29 : 28;
    }
    default -> throw new IllegalArgumentException("Invalid month: " + month);
};

System.out.println(month + "월: " + days + "일");
```

### HTTP 상태 코드

```java
int statusCode = 404;

String message = switch (statusCode) {
    case 200 -> "OK";
    case 201 -> "Created";
    case 400 -> "Bad Request";
    case 401 -> "Unauthorized";
    case 403 -> "Forbidden";
    case 404 -> "Not Found";
    case 500 -> "Internal Server Error";
    default -> "Unknown Status";
};
```

---

## 주의사항

### 1. null 처리

```java
String str = null;

// NullPointerException 발생!
switch (str) {
    case "a": ...
}

// null 체크 필요
if (str != null) {
    switch (str) { ... }
}
```

### 2. case 값은 상수만

```java
int x = 5;
final int CONST = 10;

switch (value) {
    case 1:       // OK: 리터럴
    case CONST:   // OK: final 상수
    // case x:    // 에러! 변수 불가
}
```

### 3. default 위치

```java
// default는 어디든 올 수 있지만, 마지막 권장
switch (value) {
    default:     // 가능하지만 비권장
        break;
    case 1:
        break;
}

// 권장
switch (value) {
    case 1:
        break;
    default:
        break;
}
```

---

## switch 표현식 vs 문

### 문 (Statement) - 기존 방식

```java
String result;
switch (grade) {
    case "A":
        result = "우수";
        break;
    default:
        result = "보통";
}
```

### 표현식 (Expression) - Java 12+

```java
String result = switch (grade) {
    case "A" -> "우수";
    default -> "보통";
};  // 세미콜론 필요!
```

### 표현식의 장점

- 간결한 코드
- break 필요 없음
- 값을 직접 반환
- 모든 case 처리 강제 (exhaustive)

---

> [!tip] 핵심 정리
> 
> - switch: 하나의 값을 여러 case와 비교
> - `break` 없으면 fall-through 발생
> - Java 7+: String 사용 가능
> - Java 12+: 화살표 문법, switch 표현식
> - enum과 함께 사용하면 타입 안전

---

#Java #제어문 #switch #조건문 #기초문법