# Band: Coordinated Multi-DNN Inference on Heterogeneous Mobile Processors

2023.01.18

---

## Abstract

이제는 DNN을 모바일 기기로 가져오는 많은 시도들이 이루어지고 있지만 기존의 inference framework (e.g.. Tensorflow Lite, MNN)는 single DNN을 특정 프로세서에서 돌리는 것에 집중하고 있음. 여러 DNN을 현재 framework에서 돌리는 것은 challenging한 일. 

그런 관점에서 저자들은 multi-DNN을 이기종 프로세서에서 돌릴 수 있는 BAND라는 새로운 inference system을 제안. BAND는 미리 DNN을 조사하고 subgraph들로 나눈 뒤 이 subgraph들을 프로세서에 스케줄링 하는 형태로 inference를 진행.

## Introduction

Tensorflow Lite, MNN, Mace, NCNN 등은 하나의 DNN task를 최대한 빠르게 돌리는데에 최적화가 되어 있음. 계속해서 모바일 기기에 여러 프로세서들이 포함되어 나오지만 기존의 framework들은 이들 중 가장 빠르게 수행할 수 있는 하나의 프로세서만을 선택해서 DNN task를 수행.

이와 달리 BAND는 DNN task들을 dynamically 파악하고 여러 프로세서에 효율적으로 할당하는 framework. 하지만 BAND에도 여러 challenge들이 존재하는데

1. DNN을 스케줄링 할 때 하나의 프로세서에 몰려서 ***processor contention*이 발생해서는 안됨** → 오히려 성능 저하, 반드시 이기종 프로세서들의 특성들을 잘 고려해서 스케줄링 해야 한다.
2. 모바일 기기의 프로세서들은 이질성이 크기 때문에 이들을 고려한 ***fallback operator***를 잘 다뤄야한다. 여기서 이질성이 크다는 것은 모바일 기기들이 지원하는 DNN operator가 다르다는 것. 해당 프로세서에서 실행이 불가능한 operator는 CPU로 fallback 시켜야 한다.
3. 모바일 기기의 성능의 변동성이 크다. (Dynamic Voltage and Frequency Scaling, DVFS)

이를 해결하기 위해 BAND에는 model analyzer와 scheduler가 존재. BAND는 하나의 DNN마다 여러 subgraph들의 집합들을 만들어두고 런타임 때 scheduler가 어떤 subgraph를 사용할지를 결정하여 flexibility를 보장해준다. 또, subgraph들을 만들 때 fallback operator도 고려를 하여 런타임시의 processor contention이 발생하지 않도록 한다.

## Motivation

### Utilizing Heterogeneous Processors

![Untitled](Band%20Coordinated%20Multi-DNN%20Inference%20on%20Heterogene%20c5debf11f9ce489fad2ca61d4744fb8c/Untitled.png)

앞에서 말했듯이 기존의 framework들은 GPU나 NPU 하나만 사용해서 하나의 DNN을 수행한다. 하지만 부동소수점 연산(FLOPs)에 있어 GPU가 de-facto인 server/cloud와는 달리 모바일은 de-facto인 프로세서가 없기 때문에 최대한 여러 프로세서들을 활용하는 것이 좋다.

또 하나 생각해볼 수 있는 것은 서버에 올려놓고(offloading) 서버에서 연산을 진행한 뒤 이를 네트워크를 통해서 받아오는 것이지만 bandwidth 등 다양한 이슈로 인해서 latency가 증가할 수 있으므로 offloading은 본 논문에서 제외.

### Challenges in Coordination

1. Processor Contention
    
    여러 DNN을 하나의 프로세서에서 돌리려 할 때 발생하는 processor contention. 
    
    → A coordinated runtime that utilizes heterogenyeous mobile processors must avoid processor contention between multiple DNNs.
    
2. Contention from Fallbacks
    
    ![Untitled](Band%20Coordinated%20Multi-DNN%20Inference%20on%20Heterogene%20c5debf11f9ce489fad2ca61d4744fb8c/Untitled%201.png)
    
    아직 여러 프로세서를 지원하지 않는 DNN이 존재하기 때문에 이들을 실행 가능한 프로세서로 옮겨주는 fallback operator가 필요. 하지만 이들을 그냥 fallback 하게 되면 하나의 프로세서에 fallback이 몰려서 오히려 idle한 프로세서가 생길 수 있음. 이를 잘 고려해서 스케줄링을 해야 함.
    
    추가로 현재 존재하는 framework들은 모두 CPU로 fallback을 진행함. 
    
    → Fallback operators must be carefully scheduled to boost schedulability and processor utilization
    
3. Uncerainties in Performance
    
    모바일 칩들은 System on a Chip(SoC)에 DVFS를 추가한 형태로 설계되어 성능과 전력 소비 간의 균형을 맞추는 형식으로 돌아간다. 그래서 런타임에 성능을 예측하기가 쉽지 않다. 이를 핸들하는 한가지 방법은 프로세서에 preemption을 구현하는 것인데 모바일 기기에서 이는 쉽지 않다.
    
    → Uncertainties in mobile platforms must be carefully handled with non-preemptible processors when coordinating multi-DNN workloads
    

## BAND: Subgraph-Centric Coordination

### System Overview

![Untitled](Band%20Coordinated%20Multi-DNN%20Inference%20on%20Heterogene%20c5debf11f9ce489fad2ca61d4744fb8c/Untitled%202.png)

1. 클라이언트가 RegisterModel을 통해 BAND에게 실행할 DNN 모델을 알려줌.
2. Model Analyzer가 이를 받아 모델에서 subgraph를 생성. 이때 operator들을 잘 gropuing하여 control을 잘 할 수 있도록 함.
3. 클라이언트가 RunModel을 통해 추론 요청을 보냄. 요청들은 스케줄러가 관리하는 큐에 job의 형태로 enqueue 됨. Job에는 요청한 모델과 입력 데이타와 메타 데이터 등이 있음.
4. Model Analyzer는 큐에서 job을 dequeue하고 subgraph의 정보들을 바탕으로 job의 적절한 subgraph를 선택. 
5. BAND는 각 프로세서에 대한 worker thread를 생성하여 최대 하나의 subgraph를 수행함.

### Subgraph Partitioning

### Subgraph Scheduling