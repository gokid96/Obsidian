## 입출력

```java

BufferedReader br = new BufferedReader(new InputStreamReader(System.in));

StringTokenizer st;

  

String s = br.readLine();                        // 한 줄 읽기

int n = Integer.parseInt(br.readLine());         // 숫자로 읽기

  

st = new StringTokenizer(br.readLine());         // 공백 기준 분리 (split보다 빠름)

int a = Integer.parseInt(st.nextToken());

int b = Integer.parseInt(st.nextToken());

  

StringBuilder sb = new StringBuilder();          // 출력용

sb.append(result).append("\n");

System.out.print(sb);

```
---
## 배열 입력
```java

// 1차원 배열

int[] arr = new int[n];

st = new StringTokenizer(br.readLine());

for (int i = 0; i < n; i++) {

    arr[i] = Integer.parseInt(st.nextToken());

}

  

// 2차원 배열

int[][] arr = new int[n][m];

for (int i = 0; i < n; i++) {

    st = new StringTokenizer(br.readLine());

    for (int j = 0; j < m; j++) {

        arr[i][j] = Integer.parseInt(st.nextToken());

    }

}

```
---
## 문자열 (String)

```java

s.length()              // 길이

s.charAt(i)             // i번째 문자

s.substring(1, 4)       // 1~3번째 잘라내기

s.substring(2)          // 2번째부터 끝까지

s.split(" ")            // 공백 기준 나누기 → 배열

s.contains("ab")        // 포함 여부

s.equals("abc")         // 같은지 비교 (== 쓰지 말것!)

s.indexOf("a")          // 첫 등장 위치 (없으면 -1)

s.replace("a", "b")     // a를 b로 교체

s.toUpperCase()         // 대문자로

s.toLowerCase()         // 소문자로

s.trim()                // 앞뒤 공백 제거

s.toCharArray()         // char 배열로 변환

```

---
## 문자 (char)
```java

Character.isLowerCase(c)   // 소문자인지

Character.isUpperCase(c)   // 대문자인지

Character.isDigit(c)       // 숫자인지

Character.toLowerCase(c)   // 소문자로

Character.toUpperCase(c)   // 대문자로

  

char c = '7';

int n = c - '0';           // char → int (7)

```
---
## 숫자/타입 변환
```java

Integer.parseInt("123")       // String → int

Double.parseDouble("1.5")     // String → double

String.valueOf(123)           // int → String

(int) 3.7                     // double → int (소수점 버림)

```

---
## 진법 변환
```java

// 10진수 → N진수

Integer.toBinaryString(10)    // "1010" (2진수)

Integer.toOctalString(10)     // "12" (8진수)

Integer.toHexString(10)       // "a" (16진수)

  

// N진수 → 10진수

Integer.parseInt("1010", 2)   // 10

Integer.parseInt("a", 16)     // 10

```

---
## 배열
```java

int[] arr = new int[5];

arr.length                    // 길이

Arrays.sort(arr)              // 오름차순 정렬

Arrays.fill(arr, 0)           // 전체 0으로 채우기

Arrays.toString(arr)          // 출력용 문자열

  

// 내림차순 (Integer[] 필요)

Integer[] arr2 = {3, 1, 2};

Arrays.sort(arr2, Collections.reverseOrder());

  

// 2차원 배열 정렬 (두 번째 원소 기준)

Arrays.sort(arr, (a, b) -> a[1] - b[1]);

```
---
## ArrayList
```java

ArrayList<Integer> list = new ArrayList<>();

list.add(10)            // 추가

list.get(i)             // i번째 값

list.set(i, 20)         // i번째 값 변경

list.remove(i)          // i번째 삭제

list.size()             // 크기

list.contains(10)       // 포함 여부

Collections.sort(list)  // 정렬

```
---
## HashMap
```java

HashMap<String, Integer> map = new HashMap<>();

map.put("a", 1)                  // 추가

map.get("a")                     // 값 가져오기

map.getOrDefault("b", 0)         // 없으면 기본값

map.containsKey("a")             // 키 존재 여부

map.remove("a")                  // 삭제

map.keySet()                     // 모든 키

map.values()                     // 모든 값

```
---
## Stack
```java

Stack<Character> stack = new Stack<>();

stack.push('a')       // 넣기

stack.pop()           // 빼기

stack.peek()          // 맨 위 확인 (안 빼고)

stack.isEmpty()       // 비었는지

```
---
## 수학

  

```java

Math.max(a, b)       // 큰 값

Math.min(a, b)       // 작은 값

Math.abs(a)          // 절대값

Math.pow(2, 3)       // 거듭제곱 (8.0)

Math.sqrt(16)        // 제곱근 (4.0)

Math.ceil(3.2)       // 올림 (4.0)

Math.floor(3.8)      // 내림 (3.0)

Math.round(3.5)      // 반올림 (4)

```

  

---

  

## StringBuilder

  

```java

StringBuilder sb = new StringBuilder();

sb.append("abc")       // 뒤에 추가

sb.insert(0, "x")      // 특정 위치에 삽입

sb.delete(0, 2)        // 구간 삭제

sb.reverse()           // 뒤집기

sb.toString()          // String으로 변환

```

```java 예제

import java.io.*;

import java.util.*;

  

class Main {

    public static void main(String[] args) throws Exception {

        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));

        int n = Integer.parseInt(br.readLine());

        int[][] arr = new int[n][2];

        for (int i = 0; i < n; i++) {

            StringTokenizer st = new StringTokenizer(br.readLine());

            arr[i][0] = Integer.parseInt(st.nextToken()); // 시작

            arr[i][1] = Integer.parseInt(st.nextToken()); // 종료

        }

        // 종료 시간 기준 정렬

        Arrays.sort(arr, (a, b) -> a[1] - b[1]);

        int count = 0;

        int lastEnd = -2; // 처음엔 아무거나 선택 가능하게

        for (int i = 0; i < n; i++) {

            if (arr[i][0] >= lastEnd + 2) { // 시작 >= 이전종료 + 2

                count++;

                lastEnd = arr[i][1];

            }

        }

        System.out.println(count);

    }

}

```