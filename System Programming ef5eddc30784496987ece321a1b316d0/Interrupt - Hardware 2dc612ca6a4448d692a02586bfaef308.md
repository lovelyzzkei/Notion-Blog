# Interrupt - Hardware

2023.01.17

---

## Introduction

OS는 많은 interrupt를 받는다. 키보드, usb 등 수많은 하드웨어가 OS와 통신하기 위하여 많은 interrupt를 보낸다. 이번 포스팅에서는 어떻게 **예측하지 못한 *event*를 효율적으로 처리**하는지 알아보자. 

## Process Context & Interrupt Context

Interrupt에는 두가지 종류가 있다. 

1. Exception (SW interrupt): CPU가 만들어내는 interrupt. 내 프로세스를 돌다가 발생하는 mode switching이 이에 속한다.
2. Interrupt (HW interrupt): I/O 장치가 아무때나 만드는 interrupt. Timer tick이 이에 속한다.

이 둘이 동작하는 방식은 다음과 같다.

![Untitled](Interrupt%20-%20Hardware%202dc612ca6a4448d692a02586bfaef308/Untitled.png)

1. 내 프로세스가 돌다가 ***System call, Fault* 등으로 *Exception*이 발생**하면 Kernel mode로 들어가서 **내 프로세스를 위한 커널 코드를 수행**한다. → 이와 같이 내 프로세스를 위한 커널의 코드가 실행되는 부분을 ***Process Context***라고 한다. Schedulable
2. 그러다가 다른 I/O 장치의 ***Hardware Interrupt***가 들어오면 커널은 **현재 실행하고 있던 나의 커널을 멈추고 이 *interrupt*를 *handle* 하기 위한 *mode*로** 들어간다. 즉, 내 프로세스와는 관련이 없는 interrupt handler의 코드가 실행되는 부분을 ***Interrupt Context***라고 한다. 

## Interrupt Handling

Interrupt Handler는 내 프로세스와 관련이 없는 커널의 코드를 돌리는 것이기에 **내 것에 피해를 줘서는 안된다.** 그래서 ***not-schedulable***이고 사용에 제약이 있다. 

1. *Cannot sleep/block*: **Block 하면 내것이 schedule out 되기에 block 하면 안된다.**
2. *No blocking lock, KMA*

→ 최대한 빠르게 interrupt를 처리하고 다시 내 프로세스로 돌아가야 하는 것이 Interrupt Handler의 큰 철학이다.

그러면 어떻게 Interrupt handler를 잘 만들 수 있을까? 어떻게 많은 interrupt를 효율적으로 handle 할 수 있을까? 기본적으로 Interrupt handler는 다음의 조건을 만족해야 한다.

1. ***Kernel Efficiency***
    
    Interrupt는 언제든 들어올 수 있기에 커널은 interrupt가 들어오면 **빠르게 해당 장치에 ACK를 보내서 또 다른 interrupt가 들어올 수 없도록 해야 하고,** ACK를 한 이후에는 **급한 interrupt를 먼저 처리하고(*top-half*)**, **조금 덜 급한 것은(*bottom-half*) 나중**에 돌릴 수 있도록 해야 한다.
    
2. ***Efficient handling of multiple I/Os***
    
    Interrupt를 처리하는 와중에 interrupt가 들어오면 ***nested manner*의 형태로 빠르게 처리**되어야 한다.
    
3. ***Kernel Synchronization***
    
    System call handler가 접근하고 있던 자료구조에 Interrupt handler도 접근할 수 있기게 이에 대한 관리도 필요하다.
    

기본적으로 I/O device에서 hardware interrupt가 발생하면 ***Interrupt Controller(IPC)*라는 하드웨어가 이를 다 모아서 CPU에 전달**하고, **CPU는 이 *interrupt*를 보고 *Kernel mode*의 *interrupt context*를 실행시켜 *interrupt*를 처리**한다. 이렇게 크게 ***Hardware***와 ***Software*** 부분으로 나눌 수 있는데 각 부분에 대해 조금 더 자세히 살펴보자.

## Hardware

![Untitled](Interrupt%20-%20Hardware%202dc612ca6a4448d692a02586bfaef308/Untitled%201.png)

![Untitled](Interrupt%20-%20Hardware%202dc612ca6a4448d692a02586bfaef308/Untitled%202.png)

I/O 장치의 Interrupt를 수합하는 하드웨어가 바로 위의 사진의 ***Programmable Interrupt Controller(PIC)*** 이다. 이 PIC에는 Interrupt(signal)를 감지하는 ***IRQ(Interrupt ReQuest) line***이 있어서 interrupt가 들어오면

1. 그 signal에 해당하는 **벡터로 전환**하고
2. 그 벡터를 PIC의 I/O port에 저장하고 
3. ***INTR pin*을 통해 CPU에 *interrupt*가 있다는 *signal*을 보내면**
4. **CPU는 *I/O port*에 있는 벡터를 가져가고** **잘 받았다는 ACK를 PIC에 보낸다.**

그러면 이제 CPU의 관점에서 **이 *Interrupt*를 받을지 말지 제어**할 수 있다.

1. ***Maskable*** interrupts: **감출 수 있는 *Interrupt*.** CPU register 중에서 ***Eflag***를 적절히 설정해서 I/O 장치를 통해서 들어오는 모든 interrupt에 대한 수신 여부를 제어할 수 있다. ***INTR pin*으로 들어오는 *Interrupt***가 이에 속한다.
2. ***Non-maskable*** interrupts: **CPU가 반드시 받아야만 하는 *Interrupt*.** 몇 개의 critical한 error에 대해서는 Non-maskable interrupt를 보내며 ***NMI pin*으로 들어오는 *Interrupt***가 이에 해당한다.

계속해서 하드웨어적으로 interrupt를 handling하기 위해 지원하는 architecture를 살펴보면 Interrupt Descriptor Table(IDT) 라는 것이 있다.

Physical Memory에는 ***IDT(Interrupt Descriptor Table)***이라는 것이 존재하고 **각각의 *Interrupt vector*에는 각 *Interrupt handler*의 정보가 담겨 있다.** INTR pin에 들어온 Interrupt가 몇 번째 interrupt 인지도 알고 있다. 그래서 PIC를 통해 벡터가 들어오면 아래와 같은 과정을 거쳐 처리가 된다.

![Untitled](Interrupt%20-%20Hardware%202dc612ca6a4448d692a02586bfaef308/Untitled%203.png)

1. IRQ가 들어오면 PIC에서 **몇번째 벡터인지 파악**하고 해당 벡터를 IDT에서 읽는다. 
2. 벡터의 ***Segment Selector*** 값으로 GDT에서 **해당 *interrupt handler*의 *entry point*인 *Segment Descriptor*를 읽는다.**
3. Segment Descriptor의 Base addresss 값과 벡터의 ***offset*** 값을 더하면 **실제 *interrupt handler*가 존재하는 위치를 알 수 있고 해당 코드를 돌린다.** 

즉, interrupt가 들어오면 위의 과정을 거쳐 Interrupt Handler의 코드를 수행하게 된다.

그러면 이제 User Mode에서 Kernel Mode로의 Mode switching이 이루어져야 하는데 **현재 *User*의 *Context*를 모두 저장해야 한다.** 그 중에서 **하드웨어가 일부를 저장**해주는데 **현재의 *ss, esp, eflags, cs, eip*를 커널 스택에 저장해주고(*TSS*),** ***Interrupt Handler*의 *cs*와 *eip*를 가져와서 *interrupt handler*로의 *Switching*을 일부 도와준다.** 돌아올 때도 iret를 통해 이전에 저장한 것을 되돌려준다. 

→ **하드웨어가 *Mode Switching*을 조금 도와주면서 OS의 오버헤드를 줄여준다.**

그러면 이제 커널이 Interrupt Handler를 어떻게 실행하는지 다음 포스팅에서 알아보자.