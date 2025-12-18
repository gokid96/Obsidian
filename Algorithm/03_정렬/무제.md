# 정렬 (Sorting)

## 정렬이란?

데이터를 **특정 기준에 따라 순서대로 나열**하는 것. 오름차순(작은 것 → 큰 것) 또는 내림차순(큰 것 → 작은 것)으로 정렬한다.

---

## Java 기본 정렬

### 배열 정렬 (Arrays.sort)

```java
import java.util.Arrays;

int[] arr = {5, 2, 8, 1, 9};

// 오름차순 정렬
Arrays.sort(arr);  // [1, 2, 5, 8, 9]

// 내림차순 정렬 (Integer 배열 필요)
Integer[] arr2 = {5, 2, 8, 1, 9};
Arrays.sort(arr2, Collections.reverseOrder());  // [9, 8, 5, 2, 1]

// 부분 정렬 (인덱스 1~3만)
Arrays.sort(arr, 1, 4);  // [5, 1, 2, 8, 9]
```

### 리스트 정렬 (Collections.sort)

```java
import java.util.Collections;
import java.util.ArrayList;

ArrayList<Integer> list = new ArrayList<>();
list.add(5); list.add(2); list.add(8);

// 오름차순
Collections.sort(list);  // [2, 5, 8]

// 내림차순
Collections.sort(list, Collections.reverseOrder());  // [8, 5, 2]
```

---

## 정렬 시간복잡도

|알고리즘|평균|최악|비고|
|---|---|---|---|
|Arrays.sort (기본형)|O(n log n)|O(n log n)|Dual-Pivot QuickSort|
|Arrays.sort (객체)|O(n log n)|O(n log n)|TimSort|
|Collections.sort|O(n log n)|O(n log n)|TimSort|

> 💡 Java 기본 정렬은 O(n log n)으로 충분히 빠름

---

## 커스텀 정렬

### Comparable (클래스 자체에 정렬 기준 정의)

```java
class Student implements Comparable<Student> {
    String name;
    int score;
    
    Student(String name, int score) {
        this.name = name;
        this.score = score;
    }
    
    @Override
    public int compareTo(Student o) {
        return this.score - o.score;  // 점수 오름차순
        // return o.score - this.score;  // 점수 내림차순
    }
}

// 사용
Student[] students = {...};
Arrays.sort(students);  // score 기준 정렬됨
```

### Comparator (정렬 시점에 기준 정의)

```java
// 방법 1: 익명 클래스
Arrays.sort(students, new Comparator<Student>() {
    @Override
    public int compare(Student a, Student b) {
        return a.score - b.score;
    }
});

// 방법 2: 람다식 (Java 8+)
Arrays.sort(students, (a, b) -> a.score - b.score);

// 방법 3: Comparator 메서드 체이닝
Arrays.sort(students, Comparator.comparingInt(s -> s.score));
```

---

## 다중 조건 정렬

```java
// 점수 내림차순 → 이름 오름차순
Arrays.sort(students, (a, b) -> {
    if (a.score != b.score) {
        return b.score - a.score;  // 점수 내림차순
    }
    return a.name.compareTo(b.name);  // 이름 오름차순
});

// 또는 Comparator 체이닝
Arrays.sort(students, 
    Comparator.comparingInt((Student s) -> -s.score)
              .thenComparing(s -> s.name));
```

---

## 자주 쓰는 정렬 패턴

### 문자열 정렬

```java
String[] words = {"banana", "apple", "cherry"};
Arrays.sort(words);  // 사전순: [apple, banana, cherry]

// 길이순 정렬
Arrays.sort(words, (a, b) -> a.length() - b.length());
```

### 2차원 배열 정렬

```java
int[][] arr = {{3, 5}, {1, 2}, {3, 1}};

// 첫 번째 원소 기준
Arrays.sort(arr, (a, b) -> a[0] - b[0]);

// 첫 번째 같으면 두 번째 기준
Arrays.sort(arr, (a, b) -> {
    if (a[0] != b[0]) return a[0] - b[0];
    return a[1] - b[1];
});
```

---

## 정렬 관련 팁

### compare 반환값

- **음수**: 첫 번째가 앞으로
- **0**: 순서 유지
- **양수**: 두 번째가 앞으로

### 오버플로우 주의

```java
// 나쁜 예 (오버플로우 가능)
return a - b;

// 좋은 예
return Integer.compare(a, b);
```

---

## 정리

> 💡 기본 정렬 → Arrays.sort() / Collections.sort() 💡 커스텀 정렬 → Comparator 람다식 💡 정렬 시간복잡도 → O(n log n)