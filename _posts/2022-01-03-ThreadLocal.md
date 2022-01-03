---
title: "ThreadLocal"
date: 2022-01-03
tags:
- Java
- Multi Thread
toc: true
toc_sticky: true
toc_label: "ThreadLocal"
---

> 이 내용은 인프런의 김영한님 강의 "스프링 핵심 원리 - 고급편" 을 시청하고 정리한 글입니다.


## 쓰레드로컬(ThreadLocal)이란?

- 해당 쓰레드만 접근할 수 있는 특별한 저장소를 뜻한다.
- 여러 쓰레드가 하나의 변수를 참조할 때, ThreadLocal로 선언되어 있으면 각각 쓰레드의 지역변수로써 활용된다.

### 일반적인 변수 필드

여러 쓰레드가 하나의 인스턴스의 변수에 접근하면 먼저 접근한 쓰레드의 데이터가 다른 쓰레드에 의해 변경될 수 있다.

![스크린샷 2022-01-03 오후 3.09.40.png](/assets/image/threadlocal/threadlocal1.png)

![스크린샷 2022-01-03 오후 3.12.07.png](/assets/image/threadlocal/threadlocal2.png)

쓰레드A에서 userA로 저장한 데이터가 도중에 변경되었다. 
 이렇게 일반 변수 필드에 여러 쓰레드가 동시에 접근하여 상태를 변경시키면 동시성 문제가 일어나게 된다.

### 쓰레드로컬

쓰레드 로컬을 사용하면 각 쓰레드마다 별도로 내부 저장소가 생긴다.

![스크린샷 2022-01-03 오후 3.12.15.png](/assets/image/threadlocal/threadlocal3.png)

![스크린샷 2022-01-03 오후 3.12.29.png](/assets/image/threadlocal/threadlocal4.png)

![스크린샷 2022-01-03 오후 3.12.43.png](/assets/image/threadlocal/threadlocal5.png)

thread-A와 thread-B가 각각 쓰레드 로컬을 통해 변수에 접근해서 내용을 변경하더라도 각각 전용 보관소의 내용을 변경하고 가져올 수 있게 된다.

자바에서 쓰레드 로컬을 사용하려면 `java.lang.ThreadLocal`을 사용하면 된다.

## 테스트를 통한 패턴 학습

```java
@Slf4j
public class ThreadLocalService {
	
	private ThreadLocal<String> nameStore = new ThreadLocal<>();
      
	public String logic(String name) {
		log.info("저장 name={} -> nameStore={}", name, nameStore.get()); nameStore.set(name);
		sleep(1000);
		log.info("조회 nameStore={}",nameStore.get());
		return nameStore.get();
	}
	      
	private void sleep(int millis) {
		try {
				Thread.sleep(millis);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
	} 
}
```

String이던 nameStore 변수를 ThreadLocal로 변경했다.

내부 로직을 보면 저장한 후 1초간의 대기시간을 가지고 그 후 조회를 하게 된다. 이를 이용해 아래 테스트 코드에서 동시성을 유발할 것이다.

**ThreadLocal<T> 사용법**

- set(T t) : t 값 저장
- get() : 쓰레드로컬 내부에 있는 T 변수값 조회
- remove() : 쓰레드로컬 값 제거
    - 쓰레드로컬을 모두 사용하고 난 후에 remove()를 하지 않으면 Thread Pool을 이용하는 경우 해당 쓰레드를 다시 사용할 때 이전 값이 남아있어 에러를 발생시킬 수 있기 때문에 꼭 제거해줘야 한다.

```java
@Slf4j
public class ThreadLocalServiceTest {

    private ThreadLocalService service = new ThreadLocalService();

    @Test
    void threadLocal() {
        log.info("main start");
        Runnable userA = () -> service.logic("userA");
        Runnable userB = () -> service.logic("userB");

        Thread threadA = new Thread(userA);
        threadA.setName("thread-A");
        Thread threadB = new Thread(userB);
        threadB.setName("thread-B");

        threadA.start();
        sleep(100);
        threadB.start();

        sleep(2000);
        log.info("main exit");
    }

    private void sleep(int millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

threadA를 실행한 후 0.1초의 대기시간이 지나고 threadB를 시작하게 된다.
이렇게 되면 아까 언급했듯이 내부 로직에서 저장 후 1초의 대기를 가지고 조회를 하게되므로 동시성을 체크할 수 있다.

![Untitled](/assets/image/threadlocal/threadlocal6.png)

결과를 살펴보면 thread-A가 저장을 시작하고 조회하기 전에 thread-B가 저장을 시작한다.
그 후 조회를 하게 되면 각각 본인이 저장한 값을 조회하는 것을 확인 할 수 있다.

그렇다면 ThreadLocal을 사용하지 않으면 어떻게 결과가 나올까?
ThreadLocalService 필드에 ThreadLocal<String> 대신에 그냥 String으로 변경하고 테스트를 돌려보자.

![Untitled](/assets/image/threadlocal/threadlocal7.png)

thread-A에서 userA라는 값을 조회하길 기대하는데 thread-B에 의해 도중에 userB로 변경되었다.
그로 인해 두 쓰레드 모두 userB를 조회하게 되었다.
이를 방지하기 위해 ThreadLocal을 사용하는 것이다.