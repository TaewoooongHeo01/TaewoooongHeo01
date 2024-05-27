---
layout: post
title: OS - Linux Scheduling 과 nice value
category: CS
use_math: true
excerpt: "전통적인 UNIX 계열 시스템에서 nice value 는 time slice 를 결정한다. nice value 는 -20 ~ 19 의 값을 가지고 있으며, nice value 를 통해 time slice 를 결정하는 방법은 여러 가지가 있다.

기본 개념은 nice 값에 절대적인 시간 조각을 할당하는 것이다. 기본 nice 값(0)을 가진 프로세스가 100ms의 시간 조각을 받고, 가장 낮은 우선순위(+20)를 가진 프로세스가 5ms의 시간 조각을 받는 경우, 전체 CPU 시간의 거의 대부분을 높은 우선순위 프로세스가 차지하게 된다. 이는 높은 우선순위 프로세스가 월등히 많은 CPU 시간을 확보하여 효율적으로 작업을 수행할 수 있음을 의미한다.

그러나, 모든 프로세스가 낮은 우선순위를 가지고 있을 경우도 있다. 각 프로세스가 극히 짧은 시간 동안만 실행되게 되면, 각 프로세스가 5ms 동안만 실행된다. 그렇게 모든 프로세스가 매우 짧은 시간 동안 CPU를 사용한 후 대기 상태로 전환되면, 프로세스 간의 문맥 교환(Context Switching)이 자주 발생하여 오버헤드가 증가할 수 있다. 이는 시스템의 성능을 저하시킨다."
---

Linux Scheduling
Linux 의 스케줄링 살펴보기
## Linux Scheduling Through Version 2.5
Linux 는 버전 2.5 전까지 전통적인 UNIX 스케줄링 알고리즘의 변형을 사용하였다. 하지만 이 알고리즘은 SMP 시스템을 충분히 지원하지 않았다. 따라서 매우 많은 프로세스가 실행되는 시스템에서는 저조한 성능을 보였다. 

버전 2.5 부터는 `O(1)` 이라는 스케줄링 알고리즘이 사용되었다. O(1) 스케줄링은 프로세서별 실행 큐를 도입해, 프로세스 수에 관계없이 항상 일정한 시간(O(1)) 안에 프로세스를 스케줄링 할 수 있다.

각 CPU에는 활성(active) 배열과 만료(expired) 배열이라는 두 가지 배열로 구성된 실행큐가 있다. 이것을 `RQ structure` 라고 한다. 프로세스는 할당된 시간 조각(time slice)이 남아 있는 동안 실행 가능한 상태(active)로 유지된다. 모든 시간 조각을 사용하면(expired), 다른 모든 태스크가 자신의 시간 조각을 사용할 때까지 실행할 수 없게 된다.

그렇다면 왜 `O(1)` 일까? 실행 큐는 포인터 연산을 사용하여 접근된다. active 배열에 더 이상 실행할 태스크가 없으면 배열이 교환되는데(다른 포인터 연산), 이러한 동작들이 프로세스의 수가 증가해도 스케줄링 결정 시간을 항상 일정하게 유지할 수 있는 이유다.

>참고로 여기서 교환되는 것은 active 배열과 expired 배열의 역할이다. 

하지만 한계점 또한 가지고 있다. O(1) 스케줄링 알고리즘은 대규모 서버 작업(대화형 프로세스가 적고 배치 처리가 많은 환경)에는 잘 작동하지만, 대화형 프로세스의 응답 시간에 민감한 애플리케이션의 스케줄링 지연 문제를 해결하지 못했다. 또한 모든 프로세스가 동일하게 취급되기 때문에, 특정 유형의 프로세스에게 불리하거나 특정 작업에 대한 응답 시간이 길어질 수 있다.
## Scheduler class
각 프로세스의 특성마다 다른 스케줄링 알고리즘을 제공하기 위해 현대의 Linux 스케줄러는 모듈화 되어 있다. 이것을 `Scheduler class` 라고 한다.

스케줄러 클래스와 전통적인 UNIX 계열 시스템에서 공통적으로 이해해야 하는 컨셉이 있다.
- 프로세스 우선순위(Process Priority)
- 시간조각(Timeslice)

**프로세스 우선순위(Process Priority)**: 프로세스가 얼마나 자주 실행될 것인가를 결정한다. 우선순위는 사용자 공간에서 `nice` 값 형태로 표현된다. 이 nice 값의 범위는 -20부터 19까지이며, nice 값이 낮을수록 우선순위가 높다. nice 값이 낮은 프로세스는 CPU를 더 자주, 더 오래 사용할 기회를 얻는다. 

**시간 조각(Timeslice)**: 프로세스가 한 번에 실행될 수 있는 시간의 길이를 말한다. 우선순위가 높은 프로세스는 일반적으로 더 긴 시간 조각을 할당받게 된다.

## nice value 에 대한 고민
전통적인 UNIX 계열 시스템에서 nice value 는 time slice 를 결정한다. nice value 는 -20 ~ 19 의 값을 가지고 있으며, nice value 를 통해 time slice 를 결정하는 방법은 여러 가지가 있다.

기본 개념은 nice 값에 절대적인 시간 조각을 할당하는 것이다. 기본 nice 값(0)을 가진 프로세스가 100ms의 시간 조각을 받고, 가장 낮은 우선순위(+20)를 가진 프로세스가 5ms의 시간 조각을 받는 경우, 전체 CPU 시간의 거의 대부분을 높은 우선순위 프로세스가 차지하게 된다. 이는 높은 우선순위 프로세스가 월등히 많은 CPU 시간을 확보하여 효율적으로 작업을 수행할 수 있음을 의미한다.

그러나, 모든 프로세스가 낮은 우선순위를 가지고 있을 경우도 있다. 각 프로세스가 극히 짧은 시간 동안만 실행되게 되면, 각 프로세스가 5ms 동안만 실행된다. 그렇게 모든 프로세스가 매우 짧은 시간 동안 CPU를 사용한 후 대기 상태로 전환되면, 프로세스 간의 문맥 교환(Context Switching)이 자주 발생하여 오버헤드가 증가할 수 있다. 이는 시스템의 성능을 저하시킨다.

또한 고정적인 nice value 에 의해 실행시간이 두 배 이상 차이가 날 수도 있다. 예를 들어 nice value 0, 1 에 각각 100 과 95ms 를 매핑한다고 해보자. 이 경우엔 두 프로세스가 거의 차이가 나지 않는다. 하지만 만약 nice value 가 18, 19 인 경우? 각각 10, 5ms 로 매핑된다. 이 경우 두 프로세스의 time slice 는 두 배 이상 차이가 난다. 

마지막으로, UNIX 계열의 스케줄링 정책은 I/O-bound 프로세스를 우대하여 응답 시간을 향상시킨다. 특히, I/O 작업이 끝난 프로세스는 우선순위를 부여받아 실행 큐의 맨 앞으로 이동할 수 있으며, 이 때 해당 프로세스의 time slice가 모두 소모되었어도 바로 실행될 수 있다. 이러한 우선순위 부스트는 프로세스에게 공정하지 않은 CPU 시간을 부여할 수 있는 문제를 발생시킬 수 있다.

## Completely Fair Scheduler(CFS)
`Completely Fair Scheduler(CFS)`는 각 프로세스들이 일정 비율만큼 CPU를 사용할 수 있도록 보장받는 스케줄링 방식을 구현한다. 이는 고정된 타임 슬라이스가 아니라, 프로세스의 실행 시간 비율에 기반한 것으로, 더 공정한 자원 분배를 가능하게 한다. 내부적으로 CFS는 Red-Black Tree (Rbtree)를 사용하여, vruntime 값이 가장 작은 가장 왼쪽에 있는 리프 노드를 선택함으로써 빠르고 효율적인 스케줄링 결정을 내린다.(time slice != vruntime)

CFS 는 고정적인 time slice 가 아닌, `virtual runtime(vruntime)` 이라는 새로운 단위를 사용한다. 이 값은 스위칭 비용, in/out swap 오버헤드, 캐시 hit 비율 등을 고려해 프로세스의 실행시간을 측정한다. 

여기서 스위칭 비용은 프로세스 간 전환비용, in/out swap 오버헤드는 out->데이터를 디스크로, in->필요한 데이터는 메모리로 가져올 때 이 시간을 in/out swap 오버헤드라고 한다. 

이때 `sysctl_sched_latency(target latency)` 가 등장한다. 이전에 너무 잦은 스위칭은 공평하게 실행되도록 했지만 스위칭 비용으로 인한 오버헤드가 많이 발생했다. target latency 는 현재 실행가능한 프로세스들이 모두 최소 한번씩 실행되도록 하는 시간을 말한다. 프로세스 개수로 나눠 각 프로세스에 필요한 time slice 를 알 수 있다. 

![](https://i.imgur.com/8kgywWo.png)

각 프로세스의 time slice = target latency / runnable processes 로 결정된다. 이때 실행가능한 프로세스의 개수가 너무 많다면? time slice 가 너무 적어지고 이러면 스위칭 비용에 대한 오버헤드가 급격하게 증가된다. 따라서 최소 바운드 값이 존재한다. 

`sysctl_sched_min_granularity` 라고 하는 이 값은 time slice 가 가질 수 있는 최소 값이다. 만약 프로세스가 너무 많아서 계산된 값이 최솟값보다 작아진다면 완전히 공평해지지 않을 것이다(target latency 안에 모든 프로세스가 실행될 수 없으므로)

## Niceness levels
`Niceness level` 이란 사용자가 프로세스의 우선순위를 제어할 수 있게 해준다. -20~19 까지의 값을 가지고, 값이 작을수록 높은 우선순위를 갖는다. 

![](https://i.imgur.com/Yh8lcUl.png)
nice value 가 바뀔 때마다 10% 정도의 CPU 사용량 차이가 난다. 예를 들어 레벨이 1 올라가면 -10%, 레벨이 1 낮아지면 +10% 이런 식이다. 

![](https://i.imgur.com/0kUcXzX.png)

예를 들어 다음과 같은 두 프로세스가 있다고 해보자. 
- A 프로세스. -5 의 niceness level 을 가지고 있음
- B 프로세스. 0 의 niceness level 을 가지고 있음

A 의 무게 = 3121
B 의 무게 = 1024
총 무게 = 4145

이때 만약 `sysctl_sched_latency(target latency)` 값이 48ms 라면 다음과 같이 계산된다. 

A 의 time slice = $\frac{3121}{4145} * 48 = 36ms$

B 의 time slice = $\frac{1024}{4145} * 48 = 12ms$

위에서 nice value 에 고정적인 값을 할당함으로써 경우에 따라 실행시간이 2배 이상 차이가 나는 문제(0 vs 1, 18 vs 19) 를 해결할 수 있다. 비율로 계산하기 때문이다. 

## Calculating vruntime
CFS에서는 `vruntime`을 계산하여 프로세스의 실행 순서를 결정한다. 이 시스템은 높은 우선순위(낮은 nice level)를 가진 프로세스의 `vruntime`이 느리게 증가하도록 조정하여, 이들이 더 자주 CPU 시간을 할당받을 수 있도록 한다. 프로세스의 실제 실행 시간에 따라 `vruntime`을 조정할 때, 가장 높은 가중치를 가진 프로세스를 기준으로 각 프로세스의 가중치를 반영하여 `vruntime`을 계산한다. 이로써, 높은 우선순위를 가진 프로세스는 상대적으로 더 긴 CPU 점유 시간을 갖게 됩니다.

### References
- Operating System Concepts - Book by Abraham Silberschatz, Greg Gagne, and Peter Baer Galvin