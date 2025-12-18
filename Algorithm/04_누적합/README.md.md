# 누적합 (Prefix Sum)

## 누적합이란?

배열의 **처음부터 특정 위치까지의 합**을 미리 계산해두는 기법.

핵심: **구간 합을 O(1)에 구할 수 있게 해준다.**

---

## 왜 필요한가?

### 문제 상황

"배열에서 구간 [left, right]의 합을 구하라"를 **여러 번** 물어볼 때

````java
// 단순 방법: 매번 더하기
int sum = 0;
for (int i = left; i <= right; i++) {
    sum += arr[i];
}
// 한 번에 O(n), 쿼리가 Q개면 O(n × Q)
```

**쿼리가 10만 개, 배열이 10만 개면?**
→ 10만 × 10만 = 100억 번 연산 → **시간초과**

### 누적합 사용
미리 합을 계산해두면 → 구간 합을 **O(1)**에 구함
→ 전처리 O(n) + 쿼리 O(1) × Q = O(n + Q) → **통과**

---

## 1차원 누적합

### 개념 이해
```
원본 배열 arr = [3, 2, 5, 1, 4]
인덱스         0  1  2  3  4

누적합 prefix = [0, 3, 5, 10, 11, 15]
인덱스           0  1  2   3   4   5
                 ↑
             0번째에 0을 추가 (계산 편의상)

prefix[i] = arr[0] + arr[1] + ... + arr[i-1]
prefix[1] = arr[0] = 3
prefix[2] = arr[0] + arr[1] = 3 + 2 = 5
prefix[3] = arr[0] + arr[1] + arr[2] = 3 + 2 + 5 = 10
...
````

### 구현

````java
int[] arr = {3, 2, 5, 1, 4};
int n = arr.length;

// 누적합 배열 생성 (크기 n+1, 0번 인덱스는 0으로)
int[] prefix = new int[n + 1];
for (int i = 1; i <= n; i++) {
    prefix[i] = prefix[i - 1] + arr[i - 1];
}
// prefix = [0, 3, 5, 10, 11, 15]
```

### 구간 합 구하기
```
arr[left] ~ arr[right] 구간 합
= prefix[right + 1] - prefix[left]
```

**왜 이게 되는가?**
```
prefix[right+1] = arr[0] + arr[1] + ... + arr[right]
prefix[left]    = arr[0] + arr[1] + ... + arr[left-1]
빼면            = arr[left] + ... + arr[right]  ← 우리가 원하는 것!
````

**예시: arr[1] ~ arr[3]의 합 (= 2 + 5 + 1 = 8)**

java

````java
int sum = prefix[4] - prefix[1];  // 11 - 3 = 8 ✅
```

---

## 2차원 누적합 (심화)

### 개념
2차원 배열에서 **직사각형 영역의 합**을 빠르게 구하기
```
원본 배열:          누적합 배열:
1 2 3              1  3  6
4 5 6     →        5  12 21
7 8 9              12 27 45

prefix[i][j] = (0,0)부터 (i-1,j-1)까지의 직사각형 합
````

### 구현


```java
int[][] prefix = new int[n + 1][m + 1];
for (int i = 1; i <= n; i++) {
    for (int j = 1; j <= m; j++) {
        prefix[i][j] = arr[i-1][j-1] 
                     + prefix[i-1][j] 
                     + prefix[i][j-1] 
                     - prefix[i-1][j-1];  // 중복 빼기
    }
}
```

### 영역 합 구하기

```java
// (r1, c1) ~ (r2, c2) 영역의 합
int sum = prefix[r2+1][c2+1] 
        - prefix[r1][c2+1] 
        - prefix[r2+1][c1] 
        + prefix[r1][c1];  // 두 번 빠진 거 더하기
```

---

## 시간복잡도 비교

| 연산 | 단순 방법 | 누적합 |
|------|----------|--------|
| 전처리 | - | O(n) |
| 구간 합 1회 | O(n) | O(1) |
| 구간 합 Q회 | O(n × Q) | O(n + Q) |

---

## 정리

> 💡 **"구간 합을 여러 번 구해야 한다"** → 누적합  
> 💡 `prefix[i] = prefix[i-1] + arr[i-1]`  
> 💡 구간 합 = `prefix[right+1] - prefix[left]`  
> 💡 전처리 O(n), 쿼리 O(1)