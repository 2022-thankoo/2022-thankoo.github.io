---
title: "강한 의존관계를 분리하기까지"
layout: "post"
author: "huni"
header-style: text
tags:
- domain 
- design
- OOP
- DIP
- coupling
- cohesion
---



안녕하세요. 땡쿠팀의 백엔드 개발자 후니입니다. 이번 포스팅에서는 땡쿠팀에서 겪은 **패키지간 강한 의존을 막기위한 경험**을 소개하고자 합니다.

땡쿠팀은 **우아한 테크 코스 내에서 사용되는 쿠폰을 예약하고 사용하는 서비스**를 제공하는데요. 따라서 예약이나, 만남 일정이 수락, 생성됨에 따라 도메인간 경계를 넘어 상태를 변화하거나 객체를 생성하는 코드들이 필요합니다. 비즈니스 요구사항에 맞게 개발하다보니 자연스럽게 의존성의 흐름이나 도메인 간 결합도를 느슨하게 하는 데 집중하게 됐습니다. 덕분에 여러가지 고민과 시도를 해보았는데요. 

지금부터 패키지간 강한 의존을 끊어내기 위한 땡쿠팀의 고민과 경험을 단계적으로 설명해보겠습니다.

## 1단계. Domain이 올바른 역할을 하고 있는가?

땡쿠팀에는 다음과 같은 도메인 요구사항이 존재합니다.

>  **예약이 수락될 경우 만남 일정이 생성된다.**

가장 쉽게 도메인 요구사항을 구현할 수 있는 방법은 **application layer**에 절차적으로 코드를 작성하는 것입니다.

```java
public void updateStatus(final Long memberId,
                         final Long reservationId,
                         final ReservationStatusRequest reservationStatusRequest) {
  Member foundMember = getMemberById(memberId);
  Reservation foundReservation = reservationRepository.findById(reservationId)
    .orElseThrow(() -> new InvalidReservationException(ErrorType.NOT_FOUND_RESERVATION));
  foundReservation.updateStatus(foundMember, reservationStatusRequest.getStatus());
  if(foundReservation.isAccepted()) {
    meetingService.create(Meeting.from(foundReservation)); // 주목!!
  }
}
```

위 코드를 실행하면 기능적으로는 전혀 문제가 없습니다. 사실 이렇게 코드를 짜고 넘어가도 괜찮다고 생각합니다. 

하지만 한 가지 고민이 생깁니다. 과연 **예약**이라는 도메인이 **만남 생성**이라는 책임을 직접 갖고 있는 것이 올바른 설계일까요? 저는 아니라고 생각합니다.  특히 지금과 같은 상황은 Meeting이라는 객체가 Reservation을 파라미터로 생성되는 코드도 존재하는데요, 이런 경우에는 각 도메인은 책임을 지키지 못할 뿐더러 강하게 결합된 두 객체로 인해 유지보수에도 어려움을 겪을 것입니다. (어라, 도대체 만남이 어떻게 생성되지? 왜 Reservation 스펙이 변할 때마다 Meeting이 변하지? Meeting 스펙이 변했는데 왜 Reservation에서 수정을 해야하지? 등)

그렇다면 조금만 변경해서 `Meeting.from(foundReservation);` 이 코드를 `foundReservation`을 그대로 넘기는 형태로 변경하면 **Meeting**객체 생성의 책임을 meetingService로 이식할 수 있겠습니다.

## 2단계. 양방향 해결

하지만 여전히 `updateStatus()`코드는 **Meeting**도메인의 **application layer**를 바라보고 있고, **foundReservation**을 파라미터로 받은 **meetingService**는 또 **reservation**을 바라보고 있습니다. 이 양방향 연관관계는 지금 코드를 작성할 때는 합리적으로 보일 수 있으나, 도메인 로직이 발전하고 규모가 더욱 커질 경우 유지보수가 더더욱 어려워집니다. Meeting이 변경되면 Reservation도 변경되고 그럼 또 Meeting이 변경되는 불상사가 발생하는 것이죠. 이 문제를 **DIP**를 통해 해결할 수 있습니다.

```java
//reservation.application package
public void updateStatus(final Long memberId,
                         final Long reservationId,
                         final ReservationStatusRequest reservationStatusRequest) {
  Member foundMember = getMemberById(memberId);
  Reservation foundReservation = reservationRepository.findById(reservationId)
    .orElseThrow(() -> new InvalidReservationException(ErrorType.NOT_FOUND_RESERVATION));
  foundReservation.updateStatus(foundMember, reservationStatusRequest.getStatus());
  if(foundReservation.isAccepted()) {
    reservationMeetingCreator.create(foundReservation); // 주목!!
  }
}

// reservation.application package
public interface ReservationMeetingCreator {
  void create(Reseration reservation);
}

// meeting package
public class MeetingReserveService implements ReservationMeetingCreator {
  // create 구현체
}
```

이렇게 인터페이스를 현재 `create`를 호출하는 패키지에 두고, 그 구현체를 실제 Meeting을 생성하는 지점은 **meeting** 패키지에 두면 각 도메인의 책임을 명확히 할 뿐만 아니라 의존성 관계를 **meeting -> reservation** 단방향으로 유지할 수 있습니다. 덕분에 meeting의 수정은 meeting 패키지 내부에서 해결할 수 있게 되는 것이죠.

## 3단계. 모든 것은 객체지향?

하지만 한 가지 더 고민이 생깁니다. 

**application layer**는 영속 상태의 객체 혹은 외부 서비스와 도메인을 연결하고 하나의 서비스를 제공하기 위해 도메인 로직을 적절하게 배치하는 것이 존재의 목적입니다. 따라서 **만남 일정을 생성**하는 내용은 도메인 내부에서 예약이 승인될 때 도메인이 해결해야 하는 비즈니스 로직이 아닐까 하는 생각이 듭니다. 만약 application 계층을 사용하지 않고 도메인을 호출한다면 예약이 수락됐음에도 만남 일정이 생성되지 않는 사례가 발생하게 되는 것이죠.

이 문제 또한 두 가지 방법으로 해결할 수 있을 것 같습니다.

```java
// reservation.domain
public void updateStatus(final Member member,
                         final String status,
                         final ReservedMeetingCreator reservedMeetingCreator) {
  ReservationStatus updateReservationStatus = ReservationStatus.from(status);
  validateReservation(member, updateReservationStatus);
  validateCouponStatus();

  reservationStatus = updateReservationStatus;
  if (reservationStatus.isDeny()) {
    coupon.denied();
    return;
  }
  coupon.accepted();
  reservedMeetingCreator.create(this); // 도메인 서비스의 사용
}
```

위에서 인터페이스로 분리했던 `reservedMeetingCreator`를 도메인에서 사용하는 것이죠. 인터페이스에 생성을 요청하는 것으로 실제 작동을 하게 만듭니다. 하지만 이 방법에는 단점이 하나 존재합니다. 도메인 코드가 Spring에 의존적이게 된 것이죠. Spring 없이 실제 동작을 확인하는 것이 불가능하게 된 것입니다. (가짜 객체를 이용해 테스트를 수행할 수는 있음). 하지만 도메인 중심으로 개발하다보면 Entity나 VO로 모든 서비스를 해결하기 어려울 때가 있습니다. 그럴 때 위와 같은 방법을 사용해도 좋을 것 같습니다.

또, 이런 방법이 있겠네요.

```java
// reservation.domain
public void updateStatus(final Member member,
                         final String status) {
  ReservationStatus updateReservationStatus = ReservationStatus.from(status);
  validateReservation(member, updateReservationStatus);
  validateCouponStatus();

  reservationStatus = updateReservationStatus;
  if (reservationStatus.isDeny()) {
    coupon.denied();
    return;
  }
  coupon.accepted();
  Events.publish(new MeetingCreateEvent(this)); // 이벤트 발행
}
```

바로 이벤트를 발행하는 것입니다. 이벤트를 발행하면 패키지 간 의존관계가 분리된 환경에서 객체 지향적으로 문제를 해결하는 데 도움이 되는데요. 하지만 이 또한 프레임워크에 의존적이라는 게 단점이고, 단위테스트가 어렵습니다. 물론 이벤트와 도메인을 분리하여 각각 단위테스트 하는 것이 해결 방법이 되겠지만 이를 선호하지 않을 경우에는 다른 선택을 알아봐야 할 것 같습니다.

## 대답은 NO

하지만 위 두 가지 방법으로 도메인 중심적, 객체 중심적 개발을 하다보니 오히려 객체 지향에 늪에 빠지고 알아보기 힘든 코드를 짜는 경우가 생깁니다. 그런 경우에는 **모든 것을 객체지향으로 해결할 것인가**?라는 질문을 던져봐야 할 것 같습니다.

결국 **Spring**이라는 프레임워크를 사용하고 있고 레이어드 아키텍처로 애플리케이션을 관리하고 있다면, **application layer**에서의 절차적 코드는 거치지 않을 수가 없는 지점입니다. 그렇다면 모든 것을 객체로만 풀려 하지 말고 절차적으로 문제를 더 간단하게 해결하는 것 또한 하나의 문제 해결 방법이 될 것 같습니다. 지금은 빠르게 문제를 해결하고 새로운 무언가를 개발해야할 때이기도 하구요 ㅎㅎ.

## 정리

이번 포스팅을 간단히 정리하자면 다음과 같습니다.

1. 책임 분리를 통해 결합도를 느슨하게.
2. DIP를 이용하여 단방향 의존성을 유지하자. (양방향을 해결할 수 있다면 두 도메인을 합쳐보는 것도..)
3. 모든 것을 객체지향으로 해결하지 않아도 좋다.

** 소스 코드는 [링크](https://github.com/woowacourse-teams/2022-thankoo)에서 확인할 수 있습니다. :)

긴 글 읽어주셔서 감사합니다. 이상 땡쿠팀 백엔드 개발자 후니였습니다. :)

