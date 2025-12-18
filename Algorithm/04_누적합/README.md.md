# 누적합 (Prefix Sum)

## 누적합이란?

배열의 **처음부터 특정 위치까지의 합**을 미리 계산해두는 기법. 구간 합을 O(1)에 구할 수 있게 해준다.

---

## 왜 필요한가?

### 문제 상황

배열에서 구간 합을 여러 번 구해야 할 때

```java
// 단순 방법: 매번 더하기 → O(n) × 쿼리 수
int sum = 0;
for (int i = left; i <= right; i++) {
    sum += arr[i];
}
```

쿼리가 10만 개면 → O(n × 10만) = **시간초과**

### 누적합 사용

미리 합을 계산해두면 → O(1) × 쿼리 수 = **통과**

---

## 1차원 누적합

### 개념

```
arr    = [3, 2, 5, 1, 4]
prefix = [0, 3, 5, 10, 11, 15]
         ↑ 0번째에 0 추가 (계산 편의)

prefix[i] = arr[0] + arr[1] + ... + arr[i-1]
```

### 구현

```java
int[] arr = {3, 2, 5, 1, 4};
int n = arr.length;

// 누적합 배열 생성 (크기 n+1)
int[] prefix = new int[n + 1];
for (int i = 1; i <= n; i++) {
    prefix[i] = prefix[i - 1] + arr[i - 1];
}
// prefix = [0, 3, 5, 10, 11, 15]
```

### 구간 합 구하기

```java
// arr[left] ~ arr[right] 구간 합
int sum = prefix[right + 1] - prefix[left];

// 예: arr[1] ~ arr[3] = 2 + 5 + 1 = 8
int sum = prefix[4] - prefix[1];  // 11 - 3 = 8
```

### 공식

```
구간 [left, right]의 합 = prefix[right + 1] - prefix[left]
```

---

## 누적합 활용: 층별 누적

부녀회장 문제처럼 **누적합을 반복 적용**하는 패턴

```java
// 0층: [1, 2, 3]
// 1층: [1, 3, 6]   ← 0층의 누적합
// 2층: [1, 4, 10]  ← 1층의 누적합

int[] floor = new int[n];

// 0층 초기화
for (int i = 0; i < n; i++) {
    floor[i] = i + 1;
}

// k층까지 누적합 반복
for (int j = 1; j <= k; j++) {
    for (int q = 1; q < n; q++) {
        floor[q] += floor[q - 1];
    }
}
```

---

## 2차원 누적합 (심화)

### 개념

2차원 배열에서 직사각형 영역의 합을 빠르게 구하기

```
원본 배열:
1 2 3
4 5 6
7 8 9

누적합 배열 (왼쪽 위부터 현재까지의 합):
1  3  6
5  12 21
12 27 45
```

### 구현

```java
int[][] prefix = new int[n + 1][m + 1];

for (int i = 1; i <= n; i++) {
    for (int j = 1; j <= m; j++) {
        prefix[i][j] = arr[i-1][j-1] 
                     + prefix[i-1][j] 
                     + prefix[i][j-1] 
                     - prefix[i-1][j-1];
    }
}
```

### 영역 합 구하기

```java
// (r1, c1) ~ (r2, c2) 영역의 합
int sum = prefix[r2+1][c2+1] 
        - prefix[r1][c2+1] 
        - prefix[r2+1][c1] 
        + prefix[r1][c1];
```

---

## 누적합 vs DP

|누적합|DP|
|---|---|
|합을 미리 계산|작은 문제로 큰 문제 해결|
|특정 패턴 (구간 합)|다양한 문제에 적용|
|prefix[i] = prefix[i-1] + arr[i]|점화식이 문제마다 다름|

> 💡 누적합은 DP의 한 형태로 볼 수 있다

---

## 시간복잡도

|연산|단순 방법|누적합|
|---|---|---|
|전처리|-|O(n)|
|구간 합 1회|O(n)|O(1)|
|구간 합 Q회|O(n × Q)|O(n + Q)|

---

## 정리

> 💡 구간 합 여러 번 → 누적합 사용 💡 prefix[i] = prefix[i-1] + arr[i-1] 💡 구간 합 = prefix[right+1] - prefix[left]