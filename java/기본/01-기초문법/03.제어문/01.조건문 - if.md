# 

## 개요

- 조건에 따라 **코드 실행 여부**를 결정
- 프로그램의 흐름을 제어하는 가장 기본적인 방법

---

## if문

### 기본 문법

```java
if (조건식) {
    // 조건이 true일 때 실행
}
```

### 예시

```java
int score = 80;

if (score >= 60) {
    System.out.println("합격입니다!");
}
```

### 중괄호 생략 (비권장)

```java
// 한 줄일 때 생략 가능하지만 권장하지 않음
if (score >= 60)
    System.out.println("합격");

// 권장: 항상 중괄호 사용
if (score >= 60) {
    System.out.println("합격");
}
```

---

## if-else문

### 기본 문법

```java
if (조건식) {
    // 조건이 true일 때
} else {
    // 조건이 false일 때
}
```

### 예시

```java
int age = 17;

if (age >= 18) {
    System.out.println("성인입니다");
} else {
    System.out.println("미성년자입니다");
}
```

---

## if-else if-else문

### 기본 문법

```java
if (조건1) {
    // 조건1이 true
} else if (조건2) {
    // 조건1 false, 조건2 true
} else if (조건3) {
    // 조건1,2 false, 조건3 true
} else {
    // 모든 조건 false
}
```

### 예시: 학점 계산

```java
int score = 85;
String grade;

if (score >= 90) {
    grade = "A";
} else if (score >= 80) {
    grade = "B";
} else if (score >= 70) {
    grade = "C";
} else if (score >= 60) {
    grade = "D";
} else {
    grade = "F";
}

System.out.println("학점: " + grade);  // B
```

### 예시: 요금 계산

```java
int age = 15;
int fare;

if (age < 8) {
    fare = 0;          // 무료
} else if (age < 14) {
    fare = 500;        // 어린이
} else if (age < 20) {
    fare = 800;        // 청소년
} else if (age >= 65) {
    fare = 0;          // 경로
} else {
    fare = 1200;       // 성인
}
```

---

## 중첩 if문

### 기본 형태

```java
if (조건1) {
    if (조건2) {
        // 조건1, 조건2 모두 true
    } else {
        // 조건1 true, 조건2 false
    }
} else {
    // 조건1 false
}
```

### 예시: 로그인 검증

```java
String id = "admin";
String password = "1234";

if (id.equals("admin")) {
    if (password.equals("1234")) {
        System.out.println("로그인 성공");
    } else {
        System.out.println("비밀번호 오류");
    }
} else {
    System.out.println("아이디 오류");
}
```

### 중첩 줄이기

```java
// 중첩이 깊은 코드
if (user != null) {
    if (user.isActive()) {
        if (user.hasPermission()) {
            // 처리
        }
    }
}

// Early return으로 개선
if (user == null) return;
if (!user.isActive()) return;
if (!user.hasPermission()) return;
// 처리
```

---

## 조건식 작성

### 비교 연산자

```java
if (a == b) { }   // 같다
if (a != b) { }   // 다르다
if (a > b) { }    // 크다
if (a < b) { }    // 작다
if (a >= b) { }   // 크거나 같다
if (a <= b) { }   // 작거나 같다
```

### 논리 연산자

```java
// AND: 둘 다 만족
if (age >= 18 && hasLicense) {
    System.out.println("운전 가능");
}

// OR: 하나라도 만족
if (day.equals("토") || day.equals("일")) {
    System.out.println("주말");
}

// NOT: 부정
if (!isLoggedIn) {
    System.out.println("로그인 필요");
}
```

### 복합 조건

```java
// 범위 체크
if (score >= 0 && score <= 100) {
    System.out.println("유효한 점수");
}

// 여러 값 체크
if (grade.equals("A") || grade.equals("B") || grade.equals("C")) {
    System.out.println("합격");
}
```

---

## 문자열 비교

### equals() 사용

```java
String input = "yes";

// 올바른 방법
if (input.equals("yes")) {
    System.out.println("승인");
}

// 대소문자 무시
if (input.equalsIgnoreCase("YES")) {
    System.out.println("승인");
}
```

### null 안전 비교

```java
String str = null;

// NPE 발생!
// if (str.equals("hello")) { }

// 안전한 방법 1: null 체크
if (str != null && str.equals("hello")) { }

// 안전한 방법 2: 리터럴을 앞에
if ("hello".equals(str)) { }

// 안전한 방법 3: Objects.equals()
if (Objects.equals(str, "hello")) { }
```

---

## boolean 조건

### 직접 사용

```java
boolean isValid = true;

// 권장
if (isValid) { }
if (!isValid) { }

// 불필요한 비교 (비권장)
if (isValid == true) { }
if (isValid == false) { }
```

### 메서드 반환값

```java
String str = "Hello";

if (str.isEmpty()) {
    System.out.println("빈 문자열");
}

if (str.startsWith("He")) {
    System.out.println("He로 시작");
}

if (str.contains("ell")) {
    System.out.println("ell 포함");
}
```

---

## 실전 예제

### 윤년 판별

```java
int year = 2024;

if ((year % 4 == 0 && year % 100 != 0) || year % 400 == 0) {
    System.out.println(year + "년은 윤년입니다");
} else {
    System.out.println(year + "년은 평년입니다");
}
```

### 숫자 부호 판별

```java
int num = -5;

if (num > 0) {
    System.out.println("양수");
} else if (num < 0) {
    System.out.println("음수");
} else {
    System.out.println("영");
}
```

### 삼각형 판별

```java
int a = 3, b = 4, c = 5;

if (a + b > c && b + c > a && c + a > b) {
    System.out.println("삼각형 가능");
} else {
    System.out.println("삼각형 불가");
}
```

### 입력값 검증

```java
String input = "  ";

if (input == null || input.trim().isEmpty()) {
    System.out.println("입력값이 없습니다");
} else {
    System.out.println("입력값: " + input.trim());
}
```

---

## 주의사항

### 1. = vs ==

```java
int a = 5;

// 잘못된 코드 (대입)
// if (a = 10) { }  // 컴파일 에러

// 올바른 코드 (비교)
if (a == 10) { }
```

### 2. 실수 비교

```java
double d1 = 0.1 + 0.2;
double d2 = 0.3;

// 오류 발생 가능
if (d1 == d2) { }  // false!

// 오차 범위로 비교
if (Math.abs(d1 - d2) < 0.0001) { }
```

### 3. 조건 순서

```java
// 잘못된 순서
if (score >= 60) {
    grade = "D";
} else if (score >= 70) {  // 여기 도달 안 함!
    grade = "C";
}

// 올바른 순서: 높은 점수부터
if (score >= 90) {
    grade = "A";
} else if (score >= 80) {
    grade = "B";
}
```

---

> [!tip] 핵심 정리
> 
> - `if`: 조건이 참일 때 실행
> - `if-else`: 참/거짓 분기
> - `if-else if-else`: 다중 조건
> - 문자열 비교는 `equals()` 사용
> - 중첩이 깊어지면 Early return 고려

---

#Java #제어문 #if #조건문 #기초문법