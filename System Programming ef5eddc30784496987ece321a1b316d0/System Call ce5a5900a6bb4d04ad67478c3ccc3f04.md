# System Call

2023.01.01

---

## Introduction

리눅스는 User mode와 Kernel mode의 두 가지 모드를 가지고 돌아가는 OS이다. 우리의 프로그램들이 도는 곳은 User mode이며 리눅스 전체를 총괄하는 관리자의 느낌으로 돌아가는 곳이 Kernel mode이다. 우리의 프로그램들은 User mode의 기능 만을 가지고는 돌아갈 수 없다. 파일을 읽기도 하고(open), 그 파일에 쓰기(write)도 하는 등 시스템의 기능을 수행해야 될 때가 있다. **System Call은 이러한 때에 User mode와 Kernel mode를 이어주는 interface이다.** 

## System Call

### Advantages of System Call

System Call을 사용하면 다음의 세 가지 장점이 있다.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   

- **Easy to program**: Kernel의 기능을 구체적으로 알 필요 없이 함수만을 사용하면 된다.
- **Increasing system security**: User가 Kernel에 접근할 수 있는 방법은 System Call 하나 뿐이니 Kernel로의 비정상적인 접근을 막을 수 있다.
- **Increasing program portability:** OS가 바뀌어도 동일한 프로그램을 사용할 수 있다.

### Flow of System Call

![Untitled](System%20Call%20ce5a5900a6bb4d04ad67478c3ccc3f04/Untitled.png)

System Call의 대략적인 실행 흐름은 위의 그림과 같다. 이를 조금 더 구체적으로 설명해보면

1. User task에서 System Call 함수가 사용될 시 컴파일 과정에서 인자로 넘어온 값과 해당 System Call 함수의 고유한 번호를 레지스터에 저장한다.
2. `int 0x80`의 interrupt를 통하여 Kernel Mode로 진입한다.
3. Kernel Mode로 넘어와서 수행되는 일은 세 가지인데
    1. `SAVE_ALL`: User Mode에서 저장하지 않은 나머지 레지스터 값들을 Kernel Stack에 저장.
    2. `call *(sys_call_table)(%eax, 4)`: `eax` 값을 가지고 `sys_call_table`을 참조하여 해당되는 Kernel 내부의 System Call 함수를 수행.
    3. `RESTORE_ALL`: Kernel Stack에 저장해놓은 레지스터 값들을 다시 복원.

## *Addtitional*

- 후에 조금 더 자세하게 다루겠지만 기본적으로 Kernel은 User를 믿지 않는다. 따라서 User process에서 발생할 수 있는 모든 일에 대한 대비를 해야하는데, 그 중 하나가 System Call에 **인자로 넘어오는 값들에 대한 Validity Checking**이다. 잘못된 값이 넘어올 경우 Kernel Crash가 발생할 수 있기 때문에 이를 체크해주어야하며 보통은 해당 인자의 주소값이 `PAGE_OFFSET`보다 작은지만 체크하고 더 자세한 체크 과정은 실제로 인자를 사용할 때까지 미룬다.
- 가끔가다가 Kernel Mode에서 User Mode에서 사용하고 있는 값들을 가져와야 할 경우가 있다. 이를 위하여 `get_user(),` `copy_from_user()` 등의 함수가 정의되어 있다.