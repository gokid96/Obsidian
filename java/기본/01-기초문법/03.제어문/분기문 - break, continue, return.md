## 개요

- 반복문의 **정상적인 흐름을 변경**
- `break`: 반복문 탈출
- `continue`: 다음 반복으로 건너뜀
- `return`: 메서드 종료

---

## break

### 기본 사용

- 반복문을 **즉시 종료**
- 가장 가까운 반복문만 탈출

```java
for (int i = 0; i < 10; i++) {
    if (i == 5) {
        break;  // i가 5면 종료
    }
    System.out.println(i);
}
// 출력: 0, 1, 2, 3, 4
```

### while에서 break

```java
int i = 0;
while (true) {  // 무한 루프
    if (i >= 5) {
        break;  // 탈출
    }
    System.out.println(i);
    i++;
}
```

### switch에서 break

```java
int day = 3;
switch (day) {
    case 1:
        System.out.println("월");
        break;  // 없으면 fall-through
    case 2:
        System.out.println("화");
        break;
    default:
        System.out.println("기타");
}
```

---

## continue

### 기본 사용

- 현재 반복을 **건너뛰고** 다음 반복으로
- 반복문 자체는 계속됨

```java
for (int i = 0; i < 10; i++) {
    if (i % 2 == 0) {
        continue;  // 짝수면 건너뜀
    }
    System.out.println(i);
}
// 출력: 1, 3, 5, 7, 9
```

### while에서 continue

```java
int i = 0;
while (i < 10) {
    i++;
    if (i % 3 == 0) {
        continue;  // 3의 배수 건너뜀
    }
    System.out.println(i);
}
// 출력: 1, 2, 4, 5, 7, 8, 10
```

### for vs while에서 continue 차이

```java
// for: continue 후 증감식 실행
for (int i = 0; i < 5; i++) {
    if (i == 2) continue;
    System.out.println(i);
}  // 0, 1, 3, 4

// while: continue 전에 증감 필요!
int i = 0;
while (i < 5) {
    if (i == 2) {
        i++;  // 이게 없으면 무한 루프!
        continue;
    }
    System.out.println(i);
    i++;
}
```

---

## break vs continue

### 비교

|break|continue|
|---|---|
|반복문 **종료**|현재 반복 **건너뜀**|
|루프 탈출|다음 반복으로|
|한 번만 작동|여러 번 작동 가능|

### 예시 비교

```java
// break: 5에서 멈춤
for (int i = 0; i < 10; i++) {
    if (i == 5) break;
    System.out.print(i + " ");
}
// 출력: 0 1 2 3 4

// continue: 5만 건너뜀
for (int i = 0; i < 10; i++) {
    if (i == 5) continue;
    System.out.print(i + " ");
}
// 출력: 0 1 2 3 4 6 7 8 9
```

---

## 레이블 (Label)

### 중첩 반복문 탈출

- 기본 break/continue는 가장 가까운 반복문만 영향
- 레이블로 **특정 반복문 지정** 가능

### break with label

```java
outer:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (i == 1 && j == 1) {
            break outer;  // 바깥 for 탈출
        }
        System.out.println(i + ", " + j);
    }
}
// 출력: 0,0 → 0,1 → 0,2 → 1,0 → 종료
```

### continue with label

```java
outer:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (j == 1) {
            continue outer;  // 바깥 for의 다음 반복으로
        }
        System.out.println(i + ", " + j);
    }
}
// 출력: 0,0 → 1,0 → 2,0
```

### 레이블 없는 경우 (비교)

```java
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (j == 1) {
            break;  // 안쪽 for만 탈출
        }
        System.out.println(i + ", " + j);
    }
}
// 출력: 0,0 → 1,0 → 2,0 (각 i마다 j=0만)
```

---

## return

### 메서드 종료

```java
public void process(int n) {
    if (n < 0) {
        return;  // 메서드 즉시 종료
    }
    System.out.println("처리: " + n);
}
```

### 반복문 안에서 return

```java
public int findIndex(int[] arr, int target) {
    for (int i = 0; i < arr.length; i++) {
        if (arr[i] == target) {
            return i;  // 찾으면 즉시 반환
        }
    }
    return -1;  // 못 찾으면 -1
}
```

### return vs break

```java
// break: 반복문만 탈출, 메서드 계속
public void method1() {
    for (int i = 0; i < 10; i++) {
        if (i == 5) break;
    }
    System.out.println("반복문 후 실행됨");
}

// return: 메서드 전체 종료
public void method2() {
    for (int i = 0; i < 10; i++) {
        if (i == 5) return;
    }
    System.out.println("이건 실행 안 됨");
}
```

---

## 실전 예제

### 검색 후 처리 중단

```java
int[] arr = {3, 7, 2, 9, 5};
int target = 9;
int index = -1;

for (int i = 0; i < arr.length; i++) {
    if (arr[i] == target) {
        index = i;
        break;  // 찾으면 종료
    }
}

if (index != -1) {
    System.out.println(target + " 위치: " + index);
}
```

### 유효한 입력만 처리

```java
int[] numbers = {10, -5, 20, -3, 15};
int sum = 0;

for (int num : numbers) {
    if (num < 0) {
        continue;  // 음수는 건너뜀
    }
    sum += num;
}

System.out.println("양수 합계: " + sum);  // 45
```

### 2차원 배열 검색

```java
int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};
int target = 5;
boolean found = false;

search:
for (int i = 0; i < matrix.length; i++) {
    for (int j = 0; j < matrix[i].length; j++) {
        if (matrix[i][j] == target) {
            System.out.println("찾음: [" + i + "][" + j + "]");
            found = true;
            break search;
        }
    }
}

if (!found) {
    System.out.println("없음");
}
```

### 소수 판별

```java
public boolean isPrime(int n) {
    if (n < 2) return false;
    
    for (int i = 2; i <= Math.sqrt(n); i++) {
        if (n % i == 0) {
            return false;  // 나눠지면 소수 아님
        }
    }
    return true;
}
```

### 잘못된 데이터 필터링

```java
String[] data = {"100", "abc", "200", null, "300"};
int total = 0;

for (String item : data) {
    if (item == null) continue;
    
    try {
        total += Integer.parseInt(item);
    } catch (NumberFormatException e) {
        continue;  // 숫자 아니면 건너뜀
    }
}

System.out.println("합계: " + total);  // 600
```

---

## 패턴

### Early Exit 패턴

```java
// 조건 불만족 시 빠른 종료
public void process(String input) {
    if (input == null) return;
    if (input.isEmpty()) return;
    if (!isValid(input)) return;
    
    // 실제 처리 로직
    doProcess(input);
}
```

### 검색 패턴

```java
// 찾으면 즉시 반환
for (Item item : items) {
    if (item.matches(criteria)) {
        return item;
    }
}
return null;
```

### 필터 패턴

```java
// 조건 불만족 건너뛰기
for (Item item : items) {
    if (!item.isValid()) continue;
    if (item.isExpired()) continue;
    
    process(item);
}
```

---

## 주의사항

### 1. while에서 continue

```java
// 잘못된 예: 무한 루프
int i = 0;
while (i < 5) {
    if (i == 2) {
        continue;  // i++이 실행 안 됨!
    }
    System.out.println(i);
    i++;
}

// 올바른 예
int i = 0;
while (i < 5) {
    if (i == 2) {
        i++;  // continue 전에 증감
        continue;
    }
    System.out.println(i);
    i++;
}
```

### 2. 너무 많은 break/continue

```java
// 나쁜 예: 가독성 떨어짐
for (...) {
    if (cond1) continue;
    if (cond2) break;
    if (cond3) continue;
    // ...
}

// 좋은 예: 조건 정리
for (...) {
    if (!isProcessable(item)) continue;
    process(item);
}
```

### 3. 레이블 남용 금지

```java
// 레이블은 꼭 필요할 때만
// 가능하면 메서드 분리나 플래그 사용
```

---

## 대안 (Java 8+)

### Stream으로 대체

```java
// break 대신 findFirst
Optional<Integer> first = Arrays.stream(arr)
    .filter(n -> n > 5)
    .findFirst();

// continue 대신 filter
Arrays.stream(arr)
    .filter(n -> n > 0)  // 양수만
    .forEach(System.out::println);
```

---

> [!tip] 핵심 정리
> 
> - `break`: 반복문 **탈출**
> - `continue`: 현재 반복 **건너뜀**
> - `return`: **메서드 종료**
> - 레이블: 중첩 반복문에서 특정 반복문 지정
> - while + continue는 증감 위치 주의!

---

#Java #제어문 #break #continue #분기문 #기초문법