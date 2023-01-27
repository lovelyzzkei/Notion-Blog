# Top-Half & Bottom-Half

2023.01.24

---

## Introduction

이번 포스팅에서는 interrupt의 마지막으로 interrupt handler를 잘 돌리는 방법에 대하여 알아보자.

## Characteristics of Interrupt Handler

Interrupt가 처리되는 동안 내 프로세스는 계속 ***TASK_RUNNING*** 상태이고, ***block*이 되면 안되기에 *interrupt handling*을 빠르게, 효율적으로 해주어야 한다.** 그래야 내 프로세스가 피해를 덜 받을 수 있고, interrupt도 더 많이 받을 수 있다. 이런 생각에 근거하여 ***interrupt handler*의 *task*도 급한 것과 그렇지 않은 것으로 나눌 수 있다.**

- ***Critical***: **바로 *handling*** 해야 하며 이 handler를 돌리는 중에는 다른 interrupt도 들어와서는 안된다.
- ***Noncritical***: **바로 *handling*** 해야 하지만 다른 interrupt가 들어와도 괜찮다.
- ***Noncritical deferrable***: 위의 interrupt들 보다는 **상대적으로 나중에 처리해도 괜찮은 널널한 *interrupt*.**

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled.png)

이러한 특성에 따라 Interrupt handler 내부의 task를 ***Top-Half***와 ***Bottom-Half***로 나누었는데, ***Top-Half*는 바로 처리를 하는 *interrupt handler(critical)***이고, ***Bottom-Half*는 *Top-Half* 이외의 나머지 *interrupt*를 돌리는 *handler*이며 우리가 말하는 대부분의 *interrupt*가 여기에 해당한다.** 

Bottom-Half에는 크게 softirq, tasklet, workqueue, kernel thread가 있다. 이들은 또 각 interrupt handler의 특성에 따라 위의 그림과 같이 나눌 수 있는데 이 4가지 bottom-half에 대하여 알아보자.

## Softirq

Softirq는 ***Statically-defined***, 즉 컴파일 타임에 정의된 interrupt에 대한 interrupt handler이다. CPU에 많은 itnerrupt가 들어오는데 그 중에서 ***timer, network* 등 *timing&performance-critical*한 *interrupt*에 대해서는 커널이 따로 관리를 하여 CPU에 대한 *favor*를 주는 *bottom-half*** 이다. Interrupt Context에서만 수행되며 다른 interrupt를 허용하지만 sleep 해서는 안된다.

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%201.png)

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%202.png)

softirq는 `open_softirq()`라는 함수를 이용하여 등록할 수 있다. 본인이 timing&performance critical한 interrupt를 만든다면 앞에서 공부한 자료구조에 handler 함수 등을 정의해주고 `open_softirq()` 함수를 통해서 커널에 등록해주면 된다.

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%203.png)

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%204.png)

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%205.png)

그리고 해당 interrupt가 들어오면 `raise_softirq(nr)`를 통하여 ***bit*를 설정하여 *handle* 해야 할 *interrupt*가 있음을 알린다.** 왼쪽 코드는 타이머를 수행하는 커널의 코드이다. 이렇게 매 tick마다 타이머 interrupt가 들어오며 `raise_softirq(TIMER_SOFTIRQ)`를 통해 커널에 이를 알린다. 

Interrupt가 들어왔으면 처리를 해야 한다. `do_IRQ()`의 끝에서 `irq_exit()` 함수를 돌리며 `local_softirq_ pending()` 함수를 통해 interrupt를 돌리면서 pending된 softirq가 있는지 확인하고, 있다면 `__do_softirq()` 함수를 통하여 softirq를 처리한다. `__do_softirq()`의 흐름을 살펴보자.

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%206.png)

리눅스 6.0 버전의 `__do_softirq()` 코드 중 중요한 부분만을 서술하였다.

1. `local_softirq_pending()`을 통하여 pending된 softirq를 가져옴
2. `local_irq_enable()`을 통해 interrupt를 허용
3. Interrupt context를 계속 돌면서 Interrupt를 처리한다. 이때 nested interrupt를 허용한다. 그런데 network나 storage irq의 경우 **자기 자신이 *interrupt*를 *reactive*하는 경우가 있다.** 이 경우 softirq가 계속 쌓이게 되고 점점 유저의 프로세스는 돌지 않기에 ***max_restart* 값을 두어 *handling* 하는 횟수를 제한**한다.
4. 만약 계속 처리했는데도 ***softirq*가 남았으면 `wakeup_softirqd()`를 통해 나머지는 *kernel thread*에게 돌리도록 함**

## Tasklet

그러면 statically 말고 ***dynamically***, **런타임에 내가 만든 *bottom-half*를 커널에서 돌리고 싶을때**는 어떻게 해야 할까? 이러한 bottom-half를 위한 것이 ***Tasklet***이다. 우리가 만든 bottom-half르 등록하는 방법이며 softirq handler에서 정의한 handler 중 ***HI_SOFTIRQ*와 *TASKLET_SOFTIRQ*에 해당**한다. 역시 open_softirq로 등록하기에 **모체는 *softirq***이지만 softirq와의 차이점은 ***nested execution*을 허용하지 않는다**는 것이다. 그리고 **여러 CPU에서 동일한 *tasklet*이 동시에 여러 개가 수행될 수 없다.** 

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%207.png)

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%208.png)

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%209.png)

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%202.png)

Tasklet은 `tasklet_struct`라는 자료구조 안에 정보들을 저장하며 `*func`가 tasklet을 돌리는 interrupt handler에 해당한다. `DECLARE_TASKLET`으로 새로운 tasklet을 선언하고 해당 interrupt가 들어오면 역시 수행하는 것은 `raise_softirq_irqoff()`이다. 그리고 interrupt handler가 실행되는데 `do_softirq()`를 통해 실행되는 함수는 `tasklet_action`이라는 함수이다.

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%2010.png)

`**do_softirq()`가 *tasklet*의 *irq*를 순회할 적에 `tasklet_action()` 함수를 통하여 *tasklet list*에 있는 모든 *tasklet*을 돌면서 해당 *interrupt*가 들어온 상황이라면 그 *bottom-half*를 수행**한다.

### Notice

![Untitled](Top-Half%20&%20Bottom-Half%202f1f8c256577449d9b8586f2edd782a7/Untitled%2011.png)

Tasklet은 오랫동안 쓰여오다가 2020년 7월을 기점으로 deprecated 되었다. 대신 threaded IRQ를 사용한다고 한다. 수업에서 이 부분은 배우지 않았어서 더 찾아봐야 할 것 같다.

## Ksoftirqd (kernel thread softirq)

이제 문제가 되는 상황은 ***softirq*가 계속해서 많이 들어오는 상황**이다. 커널은 **빠르게 기존의 프로세스로 돌아가야 되면서도, 들어오는 *softirq*에 대해서는 잘 처리를 해야한다.** 이 trade-off를 어떻게 다룰 것인가.

이 생각에서 리눅스가 제안한 방법이 **`do_softirq()`가 도든 동안 들어오는 *softirq*를 무시하고 빠르게 *User mode*로 돌아간다는 것**이다. ***Reactivate* 되는 *softirq*를 *handle* 하지 않고,** **대신 이거를 *ksoftirqd*라는 *kernel thread*가 *handle* 하게 한다.** Kernel thread 이기 때문에 ***process context***에서 돌아가며 sleep/block, schedulable 하다.

## Workqueue

마지막으로, 디바이스 드라이버를 짤 때 그 처리량이 많다면 ***bottom-half*를 *block* 시켜야 하는 상황이 있을 수도 있다**는 생각에서 `workqueue`가 등장했다. Block이 되어야 하기 때문에 ***Process context*에서 돌며 *schedulable* 하다.** 각 CPU마다 하나의 worker_thread가 있어서 이 kernel thread에서 workqueue에 정의한 handler 함수를 실행한다.