

## 개요

- `java.util.Arrays`
- 배열 조작을 위한 **유틸리티 클래스**
- 정렬, 검색, 복사, 비교, 채우기 등 제공
- 모든 메서드가 **static**

```java
import java.util.Arrays;
```

---

## 배열 출력

### toString()

```java
int[] arr = {3, 1, 4, 1, 5};
System.out.println(Arrays.toString(arr));  // [3, 1, 4, 1, 5]

String[] names = {"Kim", "Lee", "Park"};
System.out.println(Arrays.toString(names));  // [Kim, Lee, Park]
```

### deepToString() - 다차원 배열

```java
int[][] arr2d = {{1, 2}, {3, 4}};
System.out.println(Arrays.toString(arr2d));      // [[I@..., [I@...]
System.out.println(Arrays.deepToString(arr2d));  // [[1, 2], [3, 4]]
```

---

## 정렬

### sort() - 오름차순

```java
int[] arr = {5, 2, 8, 1, 9};
Arrays.sort(arr);
System.out.println(Arrays.toString(arr));  // [1, 2, 5, 8, 9]

String[] names = {"Charlie", "Alice", "Bob"};
Arrays.sort(names);
System.out.println(Arrays.toString(names));  // [Alice, Bob, Charlie]
```

### 부분 정렬

```java
int[] arr = {5, 2, 8, 1, 9};
Arrays.sort(arr, 1, 4);  // 인덱스 1~3만 정렬
System.out.println(Arrays.toString(arr));  // [5, 1, 2, 8, 9]
```

### 내림차순 (객체 배열)

```java
Integer[] arr = {5, 2, 8, 1, 9};
Arrays.sort(arr, Collections.reverseOrder());
System.out.println(Arrays.toString(arr));  // [9, 8, 5, 2, 1]

// 기본 타입은 직접 뒤집기
int[] intArr = {5, 2, 8, 1, 9};
Arrays.sort(intArr);
// 뒤집기
for (int i = 0; i < intArr.length / 2; i++) {
    int temp = intArr[i];
    intArr[i] = intArr[intArr.length - 1 - i];
    intArr[intArr.length - 1 - i] = temp;
}
```

### Comparator로 정렬

```java
String[] names = {"Kim", "Alice", "Bob"};

// 길이순 정렬
Arrays.sort(names, (a, b) -> a.length() - b.length());
System.out.println(Arrays.toString(names));  // [Kim, Bob, Alice]

// 역순
Arrays.sort(names, (a, b) -> b.compareTo(a));
```

---

## 검색

### binarySearch() - 이진 검색

```java
int[] arr = {1, 3, 5, 7, 9};  // 정렬 필수!

int idx = Arrays.binarySearch(arr, 5);
System.out.println(idx);  // 2

int notFound = Arrays.binarySearch(arr, 4);
System.out.println(notFound);  // -3 (삽입 위치: -(insertion point) - 1)
```

> [!warning] 주의 `binarySearch()`는 **정렬된 배열**에서만 사용! 정렬 안 되어 있으면 결과가 정확하지 않음

### 삽입 위치 계산

```java
int[] arr = {1, 3, 5, 7, 9};
int result = Arrays.binarySearch(arr, 4);

if (result < 0) {
    int insertionPoint = -(result + 1);  // 2
    System.out.println("삽입 위치: " + insertionPoint);
}
```

---

## 복사

### copyOf()

```java
int[] original = {1, 2, 3, 4, 5};

int[] copy = Arrays.copyOf(original, original.length);
System.out.println(Arrays.toString(copy));  // [1, 2, 3, 4, 5]

// 크기 확장
int[] bigger = Arrays.copyOf(original, 8);
System.out.println(Arrays.toString(bigger));  // [1, 2, 3, 4, 5, 0, 0, 0]

// 크기 축소
int[] smaller = Arrays.copyOf(original, 3);
System.out.println(Arrays.toString(smaller));  // [1, 2, 3]
```

### copyOfRange()

```java
int[] arr = {1, 2, 3, 4, 5};

int[] part = Arrays.copyOfRange(arr, 1, 4);  // 인덱스 1~3
System.out.println(Arrays.toString(part));  // [2, 3, 4]
```

---

## 비교

### equals()

```java
int[] arr1 = {1, 2, 3};
int[] arr2 = {1, 2, 3};
int[] arr3 = {1, 2, 4};

System.out.println(Arrays.equals(arr1, arr2));  // true
System.out.println(Arrays.equals(arr1, arr3));  // false
```

### deepEquals() - 다차원 배열

```java
int[][] arr1 = {{1, 2}, {3, 4}};
int[][] arr2 = {{1, 2}, {3, 4}};

System.out.println(Arrays.equals(arr1, arr2));      // false!
System.out.println(Arrays.deepEquals(arr1, arr2));  // true
```

### compare() (Java 9+)

```java
int[] arr1 = {1, 2, 3};
int[] arr2 = {1, 2, 4};

int result = Arrays.compare(arr1, arr2);
// 음수: arr1 < arr2
// 0: 같음
// 양수: arr1 > arr2
System.out.println(result);  // -1 (arr1이 더 작음)
```

---

## 채우기

### fill()

```java
int[] arr = new int[5];
Arrays.fill(arr, 7);
System.out.println(Arrays.toString(arr));  // [7, 7, 7, 7, 7]

// 부분 채우기
int[] arr2 = new int[5];
Arrays.fill(arr2, 1, 4, 9);  // 인덱스 1~3을 9로
System.out.println(Arrays.toString(arr2));  // [0, 9, 9, 9, 0]
```

### setAll() (Java 8+)

```java
int[] arr = new int[5];
Arrays.setAll(arr, i -> i * 2);  // 인덱스 * 2
System.out.println(Arrays.toString(arr));  // [0, 2, 4, 6, 8]

// 랜덤 값
Arrays.setAll(arr, i -> (int)(Math.random() * 100));
```

---

## 변환

### asList() - 배열 → List

```java
String[] arr = {"A", "B", "C"};
List<String> list = Arrays.asList(arr);
System.out.println(list);  // [A, B, C]

// 주의: 고정 크기! add/remove 불가
// list.add("D");  // UnsupportedOperationException!

// 수정 가능한 List 만들기
List<String> mutableList = new ArrayList<>(Arrays.asList(arr));
mutableList.add("D");  // OK
```

### stream() (Java 8+)

```java
int[] arr = {1, 2, 3, 4, 5};

// 합계
int sum = Arrays.stream(arr).sum();
System.out.println(sum);  // 15

// 평균
double avg = Arrays.stream(arr).average().orElse(0);
System.out.println(avg);  // 3.0

// 최대/최소
int max = Arrays.stream(arr).max().orElse(0);
int min = Arrays.stream(arr).min().orElse(0);

// 필터링
int[] evenNumbers = Arrays.stream(arr)
    .filter(n -> n % 2 == 0)
    .toArray();
System.out.println(Arrays.toString(evenNumbers));  // [2, 4]
```

---

## 기타 메서드

### mismatch() (Java 9+)

```java
int[] arr1 = {1, 2, 3, 4, 5};
int[] arr2 = {1, 2, 9, 4, 5};

int idx = Arrays.mismatch(arr1, arr2);
System.out.println(idx);  // 2 (첫 번째 다른 위치)

// 같으면 -1
int[] arr3 = {1, 2, 3, 4, 5};
System.out.println(Arrays.mismatch(arr1, arr3));  // -1
```

### parallelSort() - 병렬 정렬

```java
int[] largeArr = new int[1000000];
Arrays.setAll(largeArr, i -> (int)(Math.random() * 1000000));

// 대용량 배열에서 더 빠를 수 있음
Arrays.parallelSort(largeArr);
```

---

## 실전 예제

### 중복 제거

```java
int[] arr = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3};

int[] unique = Arrays.stream(arr)
    .distinct()
    .toArray();
System.out.println(Arrays.toString(unique));  // [3, 1, 4, 5, 9, 2, 6]
```

### 배열 뒤집기

```java
Integer[] arr = {1, 2, 3, 4, 5};
Collections.reverse(Arrays.asList(arr));
System.out.println(Arrays.toString(arr));  // [5, 4, 3, 2, 1]

// 기본 타입은 직접 또는 스트림
int[] intArr = {1, 2, 3, 4, 5};
int[] reversed = new int[intArr.length];
for (int i = 0; i < intArr.length; i++) {
    reversed[i] = intArr[intArr.length - 1 - i];
}
```

### 배열 회전

```java
// 오른쪽으로 k칸 회전
int[] arr = {1, 2, 3, 4, 5};
int k = 2;
int n = arr.length;

int[] rotated = new int[n];
for (int i = 0; i < n; i++) {
    rotated[(i + k) % n] = arr[i];
}
System.out.println(Arrays.toString(rotated));  // [4, 5, 1, 2, 3]
```

---

## 메서드 정리

|메서드|설명|
|---|---|
|`toString()`|배열을 문자열로|
|`deepToString()`|다차원 배열을 문자열로|
|`sort()`|오름차순 정렬|
|`binarySearch()`|이진 검색 (정렬 필수)|
|`copyOf()`|배열 복사|
|`copyOfRange()`|범위 복사|
|`equals()`|1차원 배열 비교|
|`deepEquals()`|다차원 배열 비교|
|`fill()`|특정 값으로 채우기|
|`asList()`|List로 변환|
|`stream()`|Stream 생성|

---

> [!tip] 핵심 정리
> 
> - `Arrays`: 배열 조작 유틸리티 클래스
> - `toString()`: 출력, `deepToString()`: 다차원
> - `sort()`: 정렬, `binarySearch()`: 검색 (정렬 필수!)
> - `copyOf()`: 복사, `equals()`: 비교
> - `fill()`: 채우기, `asList()`: List 변환

---

#Java #배열 #Arrays #유틸리티 #기초문법