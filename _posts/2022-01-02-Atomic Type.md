---
title: "Atomic Type"
date: 2022-01-02
tags:
    - Java
    - Atomic Type
    - Multi Thread
toc: true
toc_sticky: true
toc_label: "Atomic Type"
---

## **Atomic Access**

- **Atomicity(원자성)**을 보장하는 행위
- 트랜잭션의 원자성과 동일
- [오라클 자바](https://docs.oracle.com/javase/tutorial/essential/concurrency/atomic.html)에 나오는 설명이다.
    
    > In programming, an **atomic** action is one that effectively happens all at once.
    An atomic action cannot stop in the middle: it either happens completely, or it doesn't happen at all. No side effects of an atomic action are visible until the action is complete.
    
- 끝까지 수행되던가, 혹은 아무것도 수행되지 않아야 한다.
- 작업 단위가 분리되는 멀티 스레드 환경에서 필요한 연산에 Atomic operation을 사용한다.

## 동기화 문제 해결법

- Synchronized
- Volatile
- Atomic

### Synchronized

- 특정 스레드가 해당 블럭 전체를 lock 하기 때문에, 다른 스레드는 작업을 하지 못하고 기다리는 상황 발생 → Blocking이 되어 자원의 낭비
- 이를 방지하기 위해 Atomic 이용

### Volatile

- volatile 키워드는 Java 변수를 Main Memory에 저장하겠다라는 것을 명시하는 것이다.
- 변수의 값을 읽을 때마다 CPU 캐시에 저장된 값이 아닌 메인 메모리에서 읽어온다.
- 또한 변수의 값을 쓰기 작업할 때마다 메인 메모리에다 작성한다.
- 오직 1개의 쓰레드에서 쓰기 작업을 하고 다른 쓰레드는 읽기 작업만을 할 때 안정성이 보장되는 방법이다.

### **Atomic Type**

- 원자성을 보장하는 변수
- 멀티쓰레드 환경에서 `synchronized` 없이  동기화 문제를 해결하기 위해 고안된 방법이다.
- 내부적으로 CAS(Compare-And-Swap) 알고리즘을 사용해 lock 없이 동기화 처리를 할 수 있다.
    - CAS란?
        - 변수의 값을 변경하기 전에 기존에 가지고 있던 값이 내가 예상하던 값과 같을 경우에만 새로운 값으로 할당하는 방법
        
        ```java
        public class AtomicExample {
            
        	private int value;
            
            public boolean compareAndSwap(int oldValue, int newValue) {
                if(value == oldValue) {
                    value = newValue;
                    return true;
                }
                return false;
            }
        }
        ```
        
        - 위 코드와 같이 값을 변경하기 전에 예상값과 같은지 확인한다.
        - 멀티쓰레드, 멀티코어 환경에서 각 CPU는 메인 메모리에서 변수값을 참조하지 않고 각각의 캐시 영역에서 참조하게 된다. 이 때 메인 메모리에 저장된 값과 다른 경우 CAS 알고리즘을 사용해서 메인 메모리에 저장된 값과 캐시 메모리를 비교해서 일치하는 경우 새로운 값으로 바꾸고 아닌 경우 false를 반환하는 방법으로 CPU의 캐시에 잘못된 값을 참조하는 문제를 해결할 수 있다.
- Java에서 제공하는 Atomic Type은 이러한 CAS 하드웨어(CPU)의 도움을 받아 한 번에 하나의 스레드만 변수의 값을 변경할 수 있도록 제공하고 있다.
- 주요 Class
    - AtomicBoolean
    - AtomicInteger
    - AtomicLong
    - AtomicIntegerArray
    - AtomicDoubleArray

## **Atomic 사용**

먼저 간단한 쓰레드 테스트 환경을 만들어보자. 아래 SimpleLock 코드는 lock의 상태를 관리하는 로직이 있고 하나의 쓰레드만 lock를 얻는 것으로 의도했다.

```java
public class SimpleLock {

    private boolean locked = false;

    public boolean tryLock() {
        if (!locked) {
            for (int i = 0; i < 100000; i++) { }

            locked = true;
            return true;
        }
        return false;
    }
}
```

실행 될 때 lock을 거는 Runnable의 구현체이다.

```java
public class SimpleRunnable implements Runnable {

    private final SimpleLock simpleLock;

    public SimpleRunnable(SimpleLock simpleLock) {
        this.simpleLock = simpleLock;
    }

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " - " + simpleLock.tryLock());
    }
}
```

main에선 하나의 Lock 객체를 공유하는 20개의 쓰레드를 생성하고 실행한다.

```java
public class Main {
    public static void main(String[] args) {
        final SimpleLock simpleLock = new SimpleLock(); 

        for (int i = 0; i < 20; i++) {
            new Thread(new SimpleRunnable(simpleLock)).start();
        }
    }
}
```

실행시킨 결과는 아래와 같다.

![Untitled](/assets/image/atomictype/AtomicType1.png)

하나의 스레드만 lock을 획득한 것이 아니라 중구난방으로 이루어진다. 원자성이 충족되지 못하고 있다. 이를 Atomic을 사용해 보완해보자.

```java
public class SimpleLock {

    private AtomicBoolean locked = new AtomicBoolean();

    public boolean tryLock() {
        if (!locked.get()) {
            for (int i = 0; i < 100000; i++) { }
        }
        return locked.compareAndSet(false, true);
    }
}
```

`compareAndSet(expectedValue, newValue)` 

- CAS 알고리즘이라고 보면 된다. expectedValue와 값이 같으면 newValue로 set한다.

이렇게 변경한 후 코드를 실행시켜보자.

![Untitled](/assets/image/atomictype/AtomicType2.png)

하나의 스레드만 lock을 가진 것을 확인할 수 있다. 다만 Synchronized와 달리 제한적인 사용만 가능하다는 단점이 있다.

멀티 스레드 환경에서 원자성을 지키고 싶을 때 Atomic Type을 활용해보는 것도 좋은 방법이 될 거라고 생각한다.

## 참고

[https://zion830.tistory.com/58](https://zion830.tistory.com/58)

[https://n1tjrgns.tistory.com/244](https://n1tjrgns.tistory.com/244)

[https://nesoy.github.io/articles/2018-06/Java-volatile](https://nesoy.github.io/articles/2018-06/Java-volatile)