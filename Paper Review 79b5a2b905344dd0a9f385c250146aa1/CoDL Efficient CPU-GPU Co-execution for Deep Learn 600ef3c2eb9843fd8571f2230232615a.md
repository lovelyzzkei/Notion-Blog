# CoDL: Efficient CPU-GPU Co-execution for Deep Learning Inference on Mobile Devices

2023.01.17

[CoDL: Efficient CPU-GPU Co-execution for DL Inference on Mobile Devices](CoDL%20Efficient%20CPU-GPU%20Co-execution%20for%20Deep%20Learn%20600ef3c2eb9843fd8571f2230232615a/Untitled.pdf)

---

## Abstract

현재 딥러닝 추론 프레임워크들은 한번에 하나의 프로세서만 사용이 가능하다. 이에 동시에 여러 프로세서를 사용하는 연구가 많이 등장하고 있는데 성능 향상이 쉽지 않다. 이에 대한 큰 Challenge로는

1. Reduce data sharing overhead
2. Properly partition each operator between processors

이에 저자들은 CPU와 GPU를 동시에 사용하는 CoDL이라는 새로운 DL 추론 프레임워크를 제안. 여기서 제안하는 기술은 크게 두가지

1. *hybrid-type-friendly data sharing w/ hybrid-dimension partitioning & operator chain*
    
    → 각 프로세서에 더 효율적인 data type을 사용하여 data sharing의 overhead를 감소.
    
2. *non-linearity and concurrency-aware latency prediction*
    
    → light-weight but accurate latency predictor for each processors
    

## Introduction

DL은 이제 다양한 모바일 기기에 적용되고 있지만 그 모델들은 간단한 모델들에 한정되어 있음. YOLO의 경우 모바일 프로세서에서 돌리는데 몇 백 ms가 소요됨. 이에 대한 성능을 향상시키기 위해 모바일 기기의 이기종 프로세서들(heterogeneous process- ors)을 적극 활용하고자 하는 움직임이 많이 나타남.

모바일 기기에 SoC(System-on-Chips)로 내장된 CPU, GPU는 서버 CPU, GPU와는 다른 특징들이 존재.

1. 모바일 CPU와 GPU는 추론 성능이 비슷 → 동시에 돌릴 수 있는 가능성 존재
2. 동일한 메모리를 사용 → 데이터를 복사하지 않고 공유할 수 있는 가능성 존재

![Untitled](CoDL%20Efficient%20CPU-GPU%20Co-execution%20for%20Deep%20Learn%20600ef3c2eb9843fd8571f2230232615a/Untitled.png)

이를 이용하여 Abstract에서 얘기한 challenge들을 해결할 수 있지 않을까? 하지만 생각보다 쉽지 않다.

1. 동일한 메모리를 사용하기 때문에 data sharing으로 오버헤드를 줄일 수 있지 않을까?
    
    → Data sharing을 해야하기에 프로세서 동기화, 데이터 타입 변환 등 이외에 해줘야 할 것들이 많다. 이것들의 오버헤드가 더 크다면 data sharing의 의미가 없음.
    
2. 모바일 CPU와 GPU는 추론 성능이 비슷하니까 잘 나눠서 할당해주면 되지 않을까?
    
    → 어떻게 잘 나눌 것인가?
    

이를 해결하기 위해 저자들이 발견한 것들과 이를 CoDL에서 제안한 점은 다음과 같다.

1. Hybrid-type-friendly data sharing
    
    : 각 프로세서마다 선호하는 데이터 타입이 있다. (e.g.. Adreno GPU prefers *image* type rather than *buffer* type) 이를 적극 반영하자.
    
    - Hybrid-dimension partitioning: data sharing 오버헤드와 프로세서 활용률 간의 균형을 맞추기 위해 각 operator shape에 대한 input feature map의 최적 파티셔닝 방법을 찾아냄.
    - Operator Chain: shared data가 아닌 local data만 요구하는 operator 끼리 묶어서 chain을 만듬 → data sharing overhead를 줄임.
2. Non-linearity- and concurrency-aware latency prediction
    
    : 각 알고리즘에서 비선형성을 추출하겠다?
    

## Motivation and Analysis

[µLayer: Low Latency On-Device Inference Using Cooperative Single-Layer Acceleration and Processor-Friendly Quantization](%C2%B5Layer%20Low%20Latency%20On-Device%20Inference%20Using%20Coope%20fd0b7776992f43e2a2bd320ef378cc3a.md) 와 비교했을 때 성능상에 많은 차이가 존재. 그 중에서 가장 큰 차이를 보이는 이유는 data sharing. µLayer에서는 데이터 타입을 buffer로 가정해서 연구를 진행. 이것이 성능 하락을 불러왔다. 또, latency prediction도 linear로 가정하여 계산하여서 성능이 좋지 못함. 실제로 latency는 FLOPs와 linear의 관계가 아니다.

## CoDL Overview

![Untitled](CoDL%20Efficient%20CPU-GPU%20Co-execution%20for%20Deep%20Learn%20600ef3c2eb9843fd8571f2230232615a/Untitled%201.png)

CoDL의 세 가지 설계 원칙

1. Fully utilizing the computing capability of each processor
2. Minimizing the extra overhead
3. Best partitioning and balancing the workload among heterogeneous processors

CoDL은 offline과 online으로 나뉨

- Offline: 모든 data sharing overhead와 non-linearity latency response를 고려한 latency predictor 설계
- Online
    - Operator Partitioner → Hybrid-dimension partitioning & Operator chain
    - Operator co-executor → Synchronize the execution of operators according to the partitioning plan

## Hybrid-Type Friendly Data Sharing

### Hybrid-dimension partitioning

![Untitled](CoDL%20Efficient%20CPU-GPU%20Co-execution%20for%20Deep%20Learn%20600ef3c2eb9843fd8571f2230232615a/Untitled%202.png)

어떻게 operator의 텐서를 나누느냐에 따라 data sharing의 overhead가 차이난다. (a)와 같이 OC(Output Channel)를 나눌 경우 전체 input feature map을 공유해야 하지만 (b)와 같이 H를 나눌 경우 input feature map을 나눠서 공유하면 된다. 더 적은 data sharing overhead가 발생하는 것이다.

![Untitled](CoDL%20Efficient%20CPU-GPU%20Co-execution%20for%20Deep%20Learn%20600ef3c2eb9843fd8571f2230232615a/Untitled%203.png)

하지만 data sharing overhead가 적게 발생한다고 해서 실행시간이 감소하는 것은 아니다. 이는 H가 작아질 경우 GPU의 work group에 충분한 스레드가 주어지지 않아서 GPU가 노는 상황이 발생하기 때문이다. (Reduce processor utilization)

Hybrid-dimension partitioning은 위의 두 요소를 모두 고려한 파티셔닝 기법. Latency predictor와 결합하여 predictor에 operator setting과 파티셔닝 플랜을 주면 latency를 반환해주고, 이 latency가 가장 작은 파티셔닝 기법을 선택한다.

### Operator Chain

Operator Chain은 공유 데이터의 필요 횟수를 줄이기 위한 방법. Shared & locally 생성된 데이터만 필요한 operator들을 묶어서 chain을 형성한다. 여기서 가장 큰 challenge는 어떻게 빠르게 DL 모델에서 어느 operator를 묶을지 결정하는 것이다. 적절하게 operator를 묶어야하는데 그 이유는

1. chain의 파티셔닝 비율이 각 operator에게 이상적이지 않을 수 있다 → 성능 저하 유발 가능
2. chain이 길어질 수록 크기 통일을 위해 더 많은 패딩이 필요하여 더 많은 연산이 필요해진다 → 성능 저하 유발 가능

이를 모두 고려하여 저자들이 제안한 chain searching algorithm은 다음과 같다.

![Untitled](CoDL%20Efficient%20CPU-GPU%20Co-execution%20for%20Deep%20Learn%20600ef3c2eb9843fd8571f2230232615a/Untitled%204.png)

사실 이 부분은 이해가 잘 안된다. 일단 패스

## Non-Linearity- and Concurrency-Aware Latency Prediction

앞의 partitioning과 operator chain 기법은 모두 latency에 의존한다. 그러면 latency를 어떻게 빠르고 정확하게 계산해낼 수 있을까? 이전의 latency predictor들이 정확하지 않은 이유는 다음의 두가지 이다.

1. 이전의 연구들은 data sharing overhead를 고려하지 않음
2. 정확도와 가벼움을 동시에 잡지 못함

이에 저자들이 제안하는 latency predictor는 위의 두 가지 단점들을 해결하는 latency predictor이다.

1. Co-execution에서 발생하는 모든 data sharing overhead를 고려
2. Analytically formulating the non-linear latency response caused by platform features

### Latency composition of concurrency

Latency의 구성 요소는 다음과 같다.

$$
T = T_{trans}+T_{map}+T_{psync}+max(T^{cpu}_{comp}, T^{gpu}_{comp})
$$

![Untitled](CoDL%20Efficient%20CPU-GPU%20Co-execution%20for%20Deep%20Learn%20600ef3c2eb9843fd8571f2230232615a/Untitled%205.png)

여기서 data transformation과 data mapping에 걸리는 시간은 data size에 선형적인 관계를 띈다 → linear regression

$T_{psync}$의 경우 패턴이 없고 대신 드라이버에 의존하는 것으로 확인됨 → upper bound 설정

### Non-linearity-extracted executing latency computation