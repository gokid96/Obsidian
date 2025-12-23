

## 개요

- **정해진 횟수**만큼 반복 실행
- 반복 횟수가 명확할 때 주로 사용
- 배열, 컬렉션 순회에 활용

---

## 기본 for문

### 문법

```java
for (초기화; 조건식; 증감식) {
    // 반복 실행할 코드
}
```

### 실행 순서

```
1. 초기화 (한 번만)
2. 조건식 검사 → false면 종료
3. 실행문
4. 증감식
5. 2번으로 돌아감
```

### 예시

```java
for (int i = 0; i < 5; i++) {
    System.out.println(i);
}
// 출력: 0, 1, 2, 3, 4
```

---

## 다양한 for문 형태

### 감소 반복

```java
for (int i = 5; i > 0; i--) {
    System.out.println(i);
}
// 출력: 5, 4, 3, 2, 1
```

### 2씩 증가

```java
for (int i = 0; i < 10; i += 2) {
    System.out.println(i);
}
// 출력: 0, 2, 4, 6, 8
```

### 여러 변수 사용

```java
for (int i = 0, j = 10; i < j; i++, j--) {
    System.out.println("i=" + i + ", j=" + j);
}
// i=0,j=10 → i=1,j=9 → i=2,j=8 → i=3,j=7 → i=4,j=6
```

### 무한 루프

```java
for (;;) {
    // 무한 반복
    // break로 탈출 필요
}
```

---

## 배열과 for문

### 인덱스로 순회

```java
int[] arr = {10, 20, 30, 40, 50};

for (int i = 0; i < arr.length; i++) {
    System.out.println(arr[i]);
}
```

### 역순 순회

```java
for (int i = arr.length - 1; i >= 0; i--) {
    System.out.println(arr[i]);
}
// 출력: 50, 40, 30, 20, 10
```

### 값 수정

```java
int[] arr = {1, 2, 3, 4, 5};

for (int i = 0; i < arr.length; i++) {
    arr[i] *= 2;  // 각 요소 2배
}
// arr = {2, 4, 6, 8, 10}
```

---

## 향상된 for문 (for-each)

### 문법

```java
for (타입 변수 : 배열/컬렉션) {
    // 변수 사용
}
```

### 배열 순회

```java
int[] arr = {10, 20, 30, 40, 50};

for (int num : arr) {
    System.out.println(num);
}
```

### 컬렉션 순회

```java
List<String> names = Arrays.asList("Kim", "Lee", "Park");

for (String name : names) {
    System.out.println(name);
}
```

### 기본 for vs for-each

|상황|기본 for|for-each|
|---|---|---|
|단순 순회|△|✅|
|인덱스 필요|✅|❌|
|역순 순회|✅|❌|
|값 수정|✅|❌|
|가독성|△|✅|

```java
// 인덱스가 필요하면 기본 for
for (int i = 0; i < arr.length; i++) {
    System.out.println(i + "번째: " + arr[i]);
}

// 단순 순회면 for-each
for (int num : arr) {
    System.out.println(num);
}
```

---

## 중첩 for문

### 2차원 배열

```java
int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};

for (int i = 0; i < matrix.length; i++) {
    for (int j = 0; j < matrix[i].length; j++) {
        System.out.print(matrix[i][j] + " ");
    }
    System.out.println();
}
```

### 구구단

```java
for (int i = 2; i <= 9; i++) {
    System.out.println("=== " + i + "단 ===");
    for (int j = 1; j <= 9; j++) {
        System.out.println(i + " x " + j + " = " + (i * j));
    }
}
```

### 별 찍기

```java
// 직각삼각형
for (int i = 1; i <= 5; i++) {
    for (int j = 1; j <= i; j++) {
        System.out.print("*");
    }
    System.out.println();
}
/*
*
**
***
****
*****
*/

// 역직각삼각형
for (int i = 5; i >= 1; i--) {
    for (int j = 1; j <= i; j++) {
        System.out.print("*");
    }
    System.out.println();
}
```

---

## break와 continue

### break: 반복문 탈출

```java
for (int i = 0; i < 10; i++) {
    if (i == 5) {
        break;  // i가 5면 종료
    }
    System.out.println(i);
}
// 출력: 0, 1, 2, 3, 4
```

### continue: 다음 반복으로

```java
for (int i = 0; i < 10; i++) {
    if (i % 2 == 0) {
        continue;  // 짝수면 건너뜀
    }
    System.out.println(i);
}
// 출력: 1, 3, 5, 7, 9
```

### 레이블 (중첩 탈출)

```java
outer:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (i == 1 && j == 1) {
            break outer;  // 바깥 for문 탈출
        }
        System.out.println(i + ", " + j);
    }
}
// 0,0 → 0,1 → 0,2 → 1,0 → 종료
```

---

## 실전 예제

### 합계 구하기

```java
int sum = 0;
for (int i = 1; i <= 100; i++) {
    sum += i;
}
System.out.println("1~100 합: " + sum);  // 5050
```

### 최대값/최소값

```java
int[] arr = {34, 78, 12, 98, 45};
int max = arr[0];
int min = arr[0];

for (int i = 1; i < arr.length; i++) {
    if (arr[i] > max) max = arr[i];
    if (arr[i] < min) min = arr[i];
}

System.out.println("최대: " + max);  // 98
System.out.println("최소: " + min);  // 12
```

### 소수 판별

```java
int num = 17;
boolean isPrime = true;

for (int i = 2; i <= Math.sqrt(num); i++) {
    if (num % i == 0) {
        isPrime = false;
        break;
    }
}

System.out.println(num + (isPrime ? " 소수" : " 소수 아님"));
```

### 팩토리얼

```java
int n = 5;
int factorial = 1;

for (int i = 1; i <= n; i++) {
    factorial *= i;
}

System.out.println(n + "! = " + factorial);  // 120
```

### 문자열 뒤집기

```java
String str = "Hello";
String reversed = "";

for (int i = str.length() - 1; i >= 0; i--) {
    reversed += str.charAt(i);
}

System.out.println(reversed);  // olleH
```

### 특정 문자 개수 세기

```java
String text = "banana";
char target = 'a';
int count = 0;

for (int i = 0; i < text.length(); i++) {
    if (text.charAt(i) == target) {
        count++;
    }
}

System.out.println(target + " 개수: " + count);  // 3
```

---

## 성능 고려

### 반복 조건 최적화

```java
// 비효율: 매번 length() 호출
for (int i = 0; i < list.size(); i++) { }

// 효율: 미리 저장
int size = list.size();
for (int i = 0; i < size; i++) { }

// 또는 for-each
for (Object item : list) { }
```

### StringBuilder 사용

```java
// 비효율: 매번 새 String 생성
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;  // 느림
}

// 효율: StringBuilder
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

---

> [!tip] 핵심 정리
> 
> - 기본 for: `for (초기화; 조건; 증감)`
> - for-each: `for (타입 변수 : 배열/컬렉션)`
> - 인덱스 필요 → 기본 for
> - 단순 순회 → for-each
> - `break`: 탈출, `continue`: 건너뛰기

---

#Java #제어문 #for #반복문 #기초문법