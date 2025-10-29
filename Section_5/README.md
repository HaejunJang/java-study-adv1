# 5장 - 스레드 제어

### 5장에서는 스레드의 인터럽트에 대해 알아본다

### 목차

1. [인터럽트 시작](#인터럽트-시작)
2. [스레드 양보 - yield](#yield-이해하기)
3. [정리](#이번-장을-진행하며-정리)

# 인터럽트 시작

실행중인 스레드를 중간에 종료 시키려면 어떻게 해야할까?

먼저 boolean타입의 변수를 통해서 진행

```java
public class ThreadStopMainV1 {
    public static void main(String[] args) {
        MyTask task = new MyTask();
        Thread thread = new Thread(task, "work");
        thread.start();

        sleep(4000);
        log("작업 중단 지시 runFlag=false");
        task.runFlag = false;
    }

    static class MyTask implements Runnable {

        volatile boolean runFlag = true;

        @Override
        public void run() {
            while (runFlag) {
                log("작업중");
                sleep(3000);
            }
            log("자원 정리");
            log("자원 종료");
        }
    }
}
```

- 해당 코드에서는 runFlag라는 변수를 통해서 작업 종료를 요청한다
- 하지만 해당 코드는 중간에 work 스레드에 종료요청을 보내더라도 해당 기능이 끝난 뒤 종료가 된다

### 인터럽트?

- `sleep(), wait(), join()` 중 해당 스레드를 깨워야 할때 즉 스레드를 대기 상태에서 중단 시켜야할때 인터럽트를 사용한다
  - `sleep(), wait(), join()` 메서드의 공통점은 `InterruptedException`을 던지는 메서드들이다
- thread.interrupt();
  - 해당 메서드 즉시 바로 `InterruptedException`이 발생하는 것이 아닌 `InterruptedException`을 던지는 메서드를 호출하거나 또는 호출 중일때 예외가 발생하며 중단되는것이다

### 인터럽트를 사용해 개선하기 -v2

- 기존의 runFlag boolea타입 변수를 통한 중지가 아닌 interrupt를 사용해보자

기존의 코드 수정

- boolean타입의 flag 대신 thread.interrupt()를 사용

```java
main() {
...
thread.interrupt();
log("wort 스레드 인터럽트 상태1 = " + thread.isInterrupted());
...
}
while (true) {
	log("작업중");
	Thread.sleep(3000);
	}
} catch (InterruptedException e) {
	log("work 스레드 인터럽트 상태2 = " + Thread.currentThread().isInterrupted());
	log("interrupt message = " + e.getMessage());
	log("state = " + Thread.currentThread().getState());
	}
```

- 실행 결과는 기존의 boolean타입때와는 다르게 `interrupt()`가 발생시 바로 종료되는것을 확인할 수 있으며
- work 스레드의 상태1= true, work 스레드의 상태2 = false 를 확인가능하며
  - 인터럽트의 프래그 값이 true에서 `IntegerruptedException`이 발생한 뒤에는 false로 돌아온다

인터럽트를 진행하며 궁금점 정리

- 인터럽트상태는 스레드의 생명주기중 어디에 속하는가?
  - 인터럽트는 상태가 아닌 플래그이다
  - 따라서 상태에는 영향이 없으며 interrupt Flag값이 true/false 이다

### 인터럽트를 사용해 개선하기 -v3

- 기존 코드에서는 while(true) 다음에 sleep()에서 Interrupt값이 true일때 예외가 발생하여 멈추는 구조였다
- 이를 while문에서 처리하여 개선하자

```java
while(!Thread.isCurrentThread.isInterrupted()) {
...
}
```

`Thread.isInterrupted()` 메서드를 통해 인터럽트 상태값에 따라 종료되는것을 확인 할 수 있다

하지만 여전히 thread의 인터럽트 상태값을 확인하면 true 인것을 볼 수 있다

만약 후에 sleep()과 같은 메서드를 호출한다면 또 예외가 발생하므로 문제가 생길 수 있다

### 인터럽트를 사용해 개선하기 - v4

- 기존의 `Thread.isCurrentThread.isInterrupted()` 메서드는 해당 스레드가 interruped인지 상태값만 확인하며 값을 바꾸지 않는다
- `Thread.Interrupeted()` 를 사용하면
  - 스레드가 인터럽트 상태라면 true를 반환하고 상태를 false로 변경
  - 스레드가 인터럽트 상태가 아니라면 false를 반환하고 상태를 변경하지 않는다

따라서 기존 코드를 개선하면

```java
while(!Thread.interrupted()) {
...
}
```

를 통해서 스레드가 종료 되었을때도 인터럽트 상태값이 false로 정리 되는 것을 확인할 수 있었다

# yield 이해하기

- 특정 스레드가 크게 바쁘지 않은 상황이라면 잠시 다른 스레드에 CPU 실행 기회를 양보하여 효율을 높일 수 있다

실습에서는 3가지 방식으로 진행한다

1. 아무 중단 없이 진행
2. sleep()을 활용한 진행
3. yield()를 활용한 진행

### 여러번의 연산을 진행하는 코드의 일부분

```java
@Override
public void run() {
    for (int i=0; i<10; i++) {
        System.out.println(Thread.currentThread().getName() + " - " + i);
        //1. empty
        //sleep(1); //2. sleep
        //Thread.yield(); //3. yield
    }
}
```

- 다음과 같이 여러 연산을 하는 코드가 있고 결과를 확인해보면

1. 양보 없이 진행
   1. 하나의 스레드가 비교적 많이 연속적으로 처리되는것을 확인
2. sleep(1)을 활용
   1. 1밀리초 동안 `RUNNABLE → TIMED_WAITING → RUNNABLE` 로 변경한다 이렇게 될 경우 스케쥴링 큐에서 잠시 제외된다
   2. 하지만 이 방식의 문제점은 스레드 상태가 변하며 복잡한 과정을 거치고 양보할 스레드가 없을 경우에도 쉬어야하는 문제가 있다
3. yield()을 활용
   1. `Thread.yield()` 메서드를 통해 현재 실행중인 스레드가 CPU를 양보하며 다른 스레드가 사용가능하도록 한다
   2. yield 의 특징은 스레드의 상태가 변하지 않는다 `RUNNABLE` 상태를 유지하기 때문에 양보할 스레드가 없다면 계속 실행 가능하다

# 이번 장을 진행하며 정리

- 단순 Flag를 통한 스레드 정지는 즉시 정지가 어려웠다 → 그래서 interrupt를 사용
- sleep()과 같은 스레드(I`nterruptedException`을 발생하는)가 대기 상태일때 인터럽트를 호출받으면 즉시 깨어나 InterrupedException이 발생한다
- `Thread.isInterruped()` 와 `Thread.Interruped()` 의 차이점은 둘다 인터럽트의 상태값을 반환하지만 `Interruped()`는 상태를 다시 false로 바꾼다
- Thread.yield()와 Thread.sleep()의 차이점은
  - yield는 스레드 상태를 `RUNNABLE`로 유지 sleep은 `TIMED_WAITING` 상태
  - 양보할 스레드가 없을경우 yield는 바로 스케쥴링 큐를 통해 실행가능
