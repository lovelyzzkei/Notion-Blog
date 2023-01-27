# Process Address Space

2023.01.09

---

## Introduction

앞에서 커널이 어떻게 메모리에 매핑을 하는지 알아보았으니까 이제는 **커널의 관점에서 *User process*를 바라보자.** 커널은 유저에게 메모리를 어떻게 할당해줄 것인가? 유저가 요청하는 데로 전부 다 할당해 줄 것인가?

아니다. 유저가 커널에게 해를 끼치도록 메모리를 요청할 수도 있는데 커널이 곧이 곧대로 주면 안된다. **기본적으로 커널은 유저 프로세스를 믿지 않는다.** 그래서 커널은 유저 프로세스가 돌다가 발생할 수 있는 모든 에러에 대한 대비와 함께 메모리를 제공해주어야 한다. 그 과정을 한번 살펴보자.

## User Memory Allocation

유저 프로세스가 동적으로 메모리를 요청하면 바로 page frame을 할당하는 대신 **새로운 범위의 *linear address*를 사용할 수 있는 자격을 부여**한다. 이 linear address space를 **Memory Region**이라고 한다(*interval of linear address*). 커널이 이 region에 대한 자료구조를 관리하면서 유저 프로세스를 계속 감시한다.

그러면 프로세스는 언제 새로운 region을 얻을까?

- 프로세스가 새로 만들어졌을때 (fork())
- 다른 프로그램을 로딩할 때 (exec())
- 파일에 메모리 매핑을 수행할 때
- 유저의 스택에 데이터를 추가할 때
- 다른 프로세스와 데이터를 공유할 때
- 힙 공간을 늘릴 때

![Untitled](Process%20Address%20Space%20db3310783942438db651340c00b6af2b/Untitled.png)

위의 코드는 커널이 프로세스의 메모리 관련 정보를 담고 있는 `mm_struct` 자료구조의 일부이다. 프로세스의 page table을 알 수 있는 `pgd`도 여기에 저장되어 있는 것을 볼 수 있다. 여기서 `mmap`에 프로세스의 memory region을 `vm_area_struct`라는 자료구조를 이용하여 연결 리스트 형태로 관리한다. 조금 이따가 알아볼 것이지만 `mm_rb` 역시 동일한 프로세스의 memory region을 RB-Tree의 형태로 관리한다.

![Untitled](Process%20Address%20Space%20db3310783942438db651340c00b6af2b/Untitled%201.png)

mm_struct를 조금 더 살펴보면 위와 같은 멤버 변수들을 확인할 수 있다. 이들은 프로세스가 사용하는 memory region 중 Code, Data, Heap의 시작과 끝 주소와 Stack의 시작 주소 등을 의미한다. 

### vm_area_struct

![Untitled](Process%20Address%20Space%20db3310783942438db651340c00b6af2b/Untitled%202.png)

`vm_area_struct`는 각 memory region의 정보를 담고 있는 자료구조이다. 위의 코드는 `vm_area_struct`의 일부인데 memory region의 시작과 끝을 가리키는 `vm_start`, `vm_end`와 다음, 이전 memory region에 대한 포인터, 그리고 RB-Tree로 관리할 경우 트리의 노드를 가리키는 `vm_rb` 등이 있다. 

그러면 생각해볼 수 있는 것이 `mmap`을 돌다보면 Code, Data 영역들도 있을테니 `vm_start`와 `vm_end`가 위에서 봤던 `start_code`, `end_code`와 동일할 것이라는 것이다. 하지만 이는 반드시 맞지는 않는데 **memory region의 크기는 4의 배수**이기 때문이다(효율성). 이에 대한 자세한 답변은 아래의 링크를 참조하자.

[Why mm_struct->start_stack and vm_area_struct->start don't point to the same address?](https://stackoverflow.com/questions/27749792/why-mm-struct-start-stack-and-vm-area-struct-start-dont-point-to-the-same-add)

memory region을 관리하는 커널의 자료구조를 도식화한 그림은 아래와 같다.

![Untitled](Process%20Address%20Space%20db3310783942438db651340c00b6af2b/Untitled%203.png)

## Handling Memory Regions

이제 구현의 관점에서 봤을 때, 하나의 프로세스에 수많은 region이 있을 수 있다. 항상 커널이 봐야하는 것은 현재 PC가 valid한 region에 있느냐는 것인데 한가지 쉽게 생각할 수 있는 방법은 모든 vma의 vm_start와 vm_end를 비교하여 그 사이에 있는지를 확인하는 것이다. 하지만 이 작업은 단순히 연결 리스트로만 연결되어 있는 mmap으로는 시간이 오래 걸린다.

그래서 리눅스는 추가적으로 descriptor들을 담고 있는 RB-Tree를 두어 해당 주소를 담고 있는 region을 빠르게 찾는데 사용한다. 대신 연결 리스트는 전체 region을 스캔해야 할 때 사용한다. 또, page table의 permission과 vm_area의 vm_flag가 같아야 한다.

`do_mmap()`/`do_munmap()`을 사용하여 memory region을 할당받고 풀으며 함수 내부에서 `get_unmapped_ area()`등 여러 검사 및 작업이 이루어진다.

## Creating & Deleting User Address Space

### Creating

$$
fork(), vfork(), clone() \to do\_fork() \to copy\_mm()
$$

유저의 address space를 새로 만드는 것은 보통 새로운 프로세스 또는 스레드를 만드는 작업에서 발생한다. 이때 사용하는 fork(), clone()은 모두 내부적으로 do_fork()와 이후에 copy_mm()을 거쳐서 address space가 할당이 된다. 

![Untitled](Process%20Address%20Space%20db3310783942438db651340c00b6af2b/Untitled%204.png)

위의 코드는 `copy_mm(`)의 코드이다. 여기서 `fork()`와 `clone()`의 차이점은 인자로 넘어가는 값인데 `copy_mm()`에 `CLONE_VM`이 넘어갈 경우 실제 copy는 발생하지 않고 공유만 이루어지게 된다(`mm = oldmm`). 그렇지 않다면 `dup_mm()`을 통하여 copy가 발생한다. 

### Deleting

$$
sys\_exit(), do\_page\_fault() \to do\_exit()\to exit\_mm()
$$

`exit_mm()`을 통해서 프로세스가 가지고 있던 address space를 모두 놓는다. 이때 `mmput()` 이라는 함수를 통하여 deleting이 이루어진다.

![Untitled](Process%20Address%20Space%20db3310783942438db651340c00b6af2b/Untitled%205.png)

![Untitled](Process%20Address%20Space%20db3310783942438db651340c00b6af2b/Untitled%206.png)

![Untitled](Process%20Address%20Space%20db3310783942438db651340c00b6af2b/Untitled%207.png)

## Page Fault Exception Handling

Address translation 과정에서 MMU가 page fault exception을 발생시키며 do_page_fault()로 이를 핸들한다. 그러면 왜 page fault가 발생할까?

![Untitled](Process%20Address%20Space%20db3310783942438db651340c00b6af2b/Untitled%208.png)

1. ***Valid*하지 않은 *address space*에 접근하는 경우.** `vm_area_ struct`에서 정의되지 않은 space에 접근할 경우 kill 한다.
2. Code에 write를 하려는 시도는 적합하지 않은 경우이다. 따라서 ***permission denied***로 인해 kill 한다.
3. 정상적인 접근이다. 하지만 **내가 원하는 *page*의 *physical frame*이 존재하지 않는 경우**이다. 이 경우에는 frame을 매핑하면서 정상적으로 동작하도록 한다.

![Untitled](Process%20Address%20Space%20db3310783942438db651340c00b6af2b/Untitled%209.png)

커널은 `do_page_fault()` 함수를 통하여 유저가 invalid하게 address에 접근할 수 있는 많은 경우에 대해 예외 처리를 모두 구현해놓았다. 위의 그림에서 언급한 read-only 부분에 write을 시도하는 경우라든가, vma에 정의되지 않은 address에 접근하는 경우도 이 중 하나에 속한다. 대부분의 경우 프로세스를 죽이든가 `SIGSEGV` 시그널을 보내 segmentation fault 에러를 일으킨다.

하지만 세번째 경우와 같이 올바르게 접근했는데 frame이 없는 경우 frame을 할당해주어야 한다. 바로 **Demand paging**인 것이다. 이때는 `handle_pte_fault()`를 통하여 새로운 frame이 할당된다. 

또 하나의 경우는 write를 시도할 때 **페이지가 존재하고 *writable* 하지만 *read-only*로 마킹이 되어있는 경우**이다. 이 경우 커널은 해당 page가 ***Copy-On-Write***임을 알고 page의 copy를 진행한 뒤 write를 진행하고 permission 역시 다시 복원한다.