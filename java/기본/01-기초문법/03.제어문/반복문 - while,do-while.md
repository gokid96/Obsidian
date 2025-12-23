

## 개요

- **조건이 참인 동안** 반복 실행
- 반복 횟수가 불명확할 때 주로 사용
- `while`: 조건 먼저 검사
- `do-while`: 일단 실행 후 조건 검사

---

## while문

### 문법

```java
while (조건식) {
    // 조건이 true인 동안 반복
}
```

### 실행 순서

```
1. 조건식 검사 → false면 종료
2. 실행문
3. 1번으로 돌아감
```

### 기본 예시

```java
int i = 0;
while (i < 5) {
    System.out.println(i);
    i++;
}
// 출력: 0, 1, 2, 3, 4
```

---

## do-while문

### 문법

```java
do {
    // 최소 한 번은 실행
} while (조건식);  // 세미콜론 필수!
```

### 실행 순서

```
1. 실행문 (무조건 한 번)
2. 조건식 검사 → false면 종료
3. 1번으로 돌아감
```

### 기본 예시

```java
int i = 0;
do {
    System.out.println(i);
    i++;
} while (i < 5);
// 출력: 0, 1, 2, 3, 4
```

---

## while vs do-while

### 조건이 처음부터 false일 때

```java
int i = 10;

// while: 한 번도 실행 안 됨
while (i < 5) {
    System.out.println("while: " + i);
    i++;
}

// do-while: 한 번은 실행됨
i = 10;
do {
    System.out.println("do-while: " + i);
    i++;
} while (i < 5);
// 출력: do-while: 10
```

### 사용 시점

|while|do-while|
|---|---|
|조건 먼저 확인|최소 1회 실행 필요|
|0회 이상 반복|1회 이상 반복|
|일반적인 반복|메뉴, 입력 검증|

---

## while과 for 비교

### 동일한 동작

```java
// for문
for (int i = 0; i < 5; i++) {
    System.out.println(i);
}

// while문
int i = 0;
while (i < 5) {
    System.out.println(i);
    i++;
}
```

### 선택 기준

|for|while|
|---|---|
|반복 횟수가 명확|반복 횟수 불명확|
|배열/컬렉션 순회|조건 기반 반복|
|카운터 변수 필요|종료 조건만 필요|

```java
// for가 적합: 10번 반복
for (int i = 0; i < 10; i++) { }

// while이 적합: 조건 만족할 때까지
while (!isFinished) { }
```

---

## 무한 루프

### while 무한 루프

```java
while (true) {
    // 무한 반복
    if (종료조건) {
        break;  // 탈출
    }
}
```

### 실전 예시: 메뉴 시스템

```java
Scanner sc = new Scanner(System.in);

while (true) {
    System.out.println("1. 시작  2. 설정  3. 종료");
    int choice = sc.nextInt();
    
    if (choice == 1) {
        System.out.println("게임 시작");
    } else if (choice == 2) {
        System.out.println("설정 화면");
    } else if (choice == 3) {
        System.out.println("종료합니다");
        break;
    } else {
        System.out.println("잘못된 입력");
    }
}
```

---

## break와 continue

### break: 반복문 탈출

```java
int sum = 0;
int num = 1;

while (true) {
    sum += num;
    if (sum >= 100) {
        break;
    }
    num++;
}

System.out.println("합이 100 이상이 되는 수: " + num);
```

### continue: 다음 반복으로

```java
int i = 0;

while (i < 10) {
    i++;
    if (i % 2 == 0) {
        continue;  // 짝수면 건너뜀
    }
    System.out.println(i);  // 홀수만 출력
}
// 출력: 1, 3, 5, 7, 9
```

---

## 실전 예제

### 숫자 맞추기 게임

```java
Scanner sc = new Scanner(System.in);
int answer = (int)(Math.random() * 100) + 1;
int guess;
int count = 0;

System.out.println("1~100 사이의 숫자를 맞춰보세요!");

do {
    System.out.print("입력: ");
    guess = sc.nextInt();
    count++;
    
    if (guess > answer) {
        System.out.println("더 작은 수입니다");
    } else if (guess < answer) {
        System.out.println("더 큰 수입니다");
    }
} while (guess != answer);

System.out.println("정답! " + count + "번 만에 맞췄습니다");
```

### 입력 검증

```java
Scanner sc = new Scanner(System.in);
int age;

do {
    System.out.print("나이를 입력하세요 (0-150): ");
    age = sc.nextInt();
} while (age < 0 || age > 150);

System.out.println("입력된 나이: " + age);
```

### 숫자 자릿수 세기

```java
int num = 12345;
int count = 0;

while (num > 0) {
    num /= 10;
    count++;
}

System.out.println("자릿수: " + count);  // 5
```

### 숫자 뒤집기

```java
int num = 12345;
int reversed = 0;

while (num > 0) {
    reversed = reversed * 10 + num % 10;
    num /= 10;
}

System.out.println("뒤집은 숫자: " + reversed);  // 54321
```

### 최대공약수 (유클리드 호제법)

```java
int a = 48, b = 18;

while (b != 0) {
    int temp = b;
    b = a % b;
    a = temp;
}

System.out.println("최대공약수: " + a);  // 6
```

### 파일 읽기 패턴

```java
BufferedReader reader = new BufferedReader(new FileReader("file.txt"));
String line;

while ((line = reader.readLine()) != null) {
    System.out.println(line);
}

reader.close();
```

### 이터레이터 순회

```java
List<String> list = Arrays.asList("A", "B", "C");
Iterator<String> it = list.iterator();

while (it.hasNext()) {
    String item = it.next();
    System.out.println(item);
}
```

---

## 중첩 while

### 구구단

```java
int i = 2;
while (i <= 9) {
    int j = 1;
    while (j <= 9) {
        System.out.println(i + " x " + j + " = " + (i * j));
        j++;
    }
    i++;
}
```

### 2차원 검색

```java
int[][] matrix = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
int target = 5;
boolean found = false;

int i = 0;
while (i < matrix.length && !found) {
    int j = 0;
    while (j < matrix[i].length) {
        if (matrix[i][j] == target) {
            System.out.println("찾음: [" + i + "][" + j + "]");
            found = true;
            break;
        }
        j++;
    }
    i++;
}
```

---

## 주의사항

### 1. 무한 루프 방지

```java
// 잘못된 예: 증감 없음
int i = 0;
while (i < 5) {
    System.out.println(i);
    // i++ 없음 → 무한 루프!
}

// 올바른 예
int i = 0;
while (i < 5) {
    System.out.println(i);
    i++;  // 증감 필수
}
```

### 2. 조건 업데이트

```java
// 잘못된 예: 조건이 바뀌지 않음
boolean running = true;
while (running) {
    // running이 절대 false가 안 됨 → 무한 루프
}

// 올바른 예
boolean running = true;
while (running) {
    if (종료조건) {
        running = false;
    }
}
```

### 3. do-while 세미콜론

```java
do {
    // 코드
} while (조건);  // 세미콜론 필수!
```

---

## 패턴 정리

### 센티넬 값 사용

```java
Scanner sc = new Scanner(System.in);
int sum = 0;
int num;

System.out.println("숫자 입력 (-1 종료):");

while ((num = sc.nextInt()) != -1) {
    sum += num;
}

System.out.println("합계: " + sum);
```

### 플래그 사용

```java
boolean found = false;

while (!found) {
    // 검색 로직
    if (찾음) {
        found = true;
    }
}
```

### 카운터 제한

```java
int maxAttempts = 3;
int attempts = 0;

while (attempts < maxAttempts) {
    // 시도
    attempts++;
}
```

---

> [!tip] 핵심 정리
> 
> - `while`: 조건 먼저 검사 (0회 이상 실행)
> - `do-while`: 먼저 실행 후 조건 검사 (1회 이상 실행)
> - 반복 횟수 불명확할 때 while 사용
> - 무한 루프는 반드시 탈출 조건 필요
> - 조건 변수 업데이트 잊지 말기!

---

#Java #제어문 #while #dowhile #반복문 #기초문법