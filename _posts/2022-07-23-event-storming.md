---
title: "EventStorming"
layout: "post"
author: "skull"
header-style: text
tags:
- domain 
- design
---

# 왜 DDD를 사용하나?
1. DDD를 사용하면 실제 비즈니스 도메인을 아키텍처에 투영해 도메인을 정의하고 이 도메인을 바탕으로 커뮤니케이션 할 수 있다.

2. 도메인 전문가와 개발자들이 공통된 언어를 사용할 수 있다.

위와 같은 이점들로 인해 DDD는 현재 널리 사용되고 있는 설계 방식이다. 그리고 DDD를 하기 위한 가장 쉬운 방법 중 하나가 Event Storming이다.

# Event Storming
## 사전 준비:
1. 다양한 색의 포스트잇, 마커

2. 포스트잇을 붙일 수 있는 큰 공간(모두가 볼 수 있어야 한다)

3. 각 포스트잇 색이 의미하는 바를 명시해 놓은 공간(모두가 볼 수 있어야 한다)

4. 좌석은 없이, 서서 진행한다

5. 개발자 뿐만아니라 해당 소프트웨어를 제작하는데 필요한 모든 인원들이 함께 참석한다

## Event Storming의 장점
1. 간단한 방식으로 진행되기 때문에 이해가 쉽다.

2. 개발과 비즈니스가 생각을 맞춰볼 수 있다.

3. MSA를 디자인할 때 도메인과 서브도메인을 가를 수 있다.

## Event Storming 결과
Event Storming의 최종 결과의 예시는 다음과 같다.  
![eventStormingResult](/img/in-post/2022-07-23-event-storming/eventStormingResult.png)  
Event Storming의 최종 결과는 큰 바운더리를 여러개 가지며 하나의 바운더리 안에 여러개의 포스트잇이 존재한다. 각 포스트잇들은 위와 같이 연관된 것들 끼리 붙임으로서 상관 관계를 나타낸다.

## 각 포스트잇의 의미
위 "Event Sotrming 결과"에서 나온 각 포스트잇의 의미는 다음과 같다.
### Domain Event
이벤 트를 도출할 때 사용한다. 사용자에게 발생하는 이벤트들을 서술하며 과거형 또는 수동형으로 서술한다.

### Command

커맨드는 이벤트들을 발생시키는 행위다.

### External System

이벤트 발생에 필요한 외부 시스템.

### Aggregate

서로 관련있는 도메인 모델들의 집합이다. 한 개 이상의 entity 또는 value로 이뤄져 있다. 각 aggregate는 aggregate라는 루트 도메인을 가진다. 이 루트는 aggregate 내에 속한 객체의 변경을 책임지고, 도메인 규칙에 따라 내부에 존재하는 도메인 모델들의 일관성을 유지한다. aggregate의 예시는 다음과 같다.
```java
@entity
public class Item {
    private String name;
    private int quantity;
    private PorductId productId;
}
```
Aggregate 끼리는 각자의 id를 통해 서로 참조해야 한다. 이를 통해 결합도를 낮춘다. Aggreate 끼리 직접 참조를 하면 다음과 같은 문제가 발생한다.

1. 다른 aggregate를 수정하기 쉬워지기 때문에 우발적인 수정이 발생할 수 있다.

2. 확장이 어렵다. 모든 aggreagate를 모두 같은 곳에 저장해야 한다.

command, aggregate, event는 command가 aggregate에게 영향을 주고 최종적으로 event가 발생되는 식으로 관계가 맺어져 있다. 예시는 다음과 같다.  
![command, aggregate, event flow](/img/in-post/2022-07-23-event-storming/commandAggregateEventFlow.png)  
위 예시에서 볼 수 있듯이 aggregate와 event는 원자성을 가진다. 즉, 어떤 이벤트가 발생했을 때 aggregate의 상태에 변화가 발생한다.

## Event Storming 실행 절차

이벤트를 붙일 때는 왼쪽에서 오른쪽으로 시간 순으로 붙인다.  
![event sotrming flow1](/img/in-post/2022-07-23-event-storming/eventSormingFlow1.png)  
command는 event와 짝으로 발생한하며 각 event 왼편에 위치한다.  
![event sotrming flow2](/img/in-post/2022-07-23-event-storming/eventSormingFlow2.png)  
이제 command와 event와 관련된 aggregate를 붙인다.  
![event sotrming flow3](/img/in-post/2022-07-23-event-storming/eventSormingFlow3.png)  
command는 aggregate에 영향을 미쳐서 aggregate의 상태가 바뀌고, 그로 인해 event가 발생한다.  

## Boris diagram
Event diagram을 통해 나온 상관관계들을 aggregate 단위로 봤을 때 어떤 상호작용을 할지 볼 수 있는 다이어그램이다. 아래 예시에서 노란색은 aggregate를 나타내며 분홍색은 external system을 나타낸다.
![boris diagram](/img/in-post/2022-07-23-event-storming/borisDiagram.png)  

## Snap E
위 과정을 통해 도출한 서비스 중 일부를 필요하다면 좀 더 상세하게 적을 수 있다. 여기에는 해당 서비스가 사용하는 API, data, user story 등이 포함된다. 이를 통해 해당 서비스와 연관된 모든 것들을 한 눈에 파악할 수 있다.
![snap e](/img/in-post/2022-07-23-event-storming/snape.png)


