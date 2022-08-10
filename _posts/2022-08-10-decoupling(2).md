---
title: "강한 의존관계를 분리하기까지(2) - Separated Interface"
layout: "post"
author: "huni"
header-style: text
comments: true
tags:
- domain 
- design
- OOP
- DIP
- coupling
- cohesion
---

안녕하세요. 땡쿠팀의 백엔드 개발자 후니입니다. 지난 포스팅에 이어서 강한 의존관계를 분리하는 방법을 소개하고자 하는데요. 
`Separated Interface` 패턴을 이용하여 의존관계를 분리한 제 경험을 작성해보겠습니다.

> Separated Interface?
>
> 개발을 하다보면, 서로 다른 두 개의 시스템 파트의 결합도를 줄임으로써 설계의 수준을 개선할 수 있습니다. 일반적인 구조를 부정하고 다른 패키지를 참조해야 할 때 이 패턴을 주로 사용할 수 있는데요. 구현체를 상대 패키지에 두고, 인터페이스를 현재 패키지에 위치시키는 것입니다. 그러면 클라이언트는 구현체에 대한 정보는 참조할 필요 없이 현재 패키지의 인터페이스만 참조하게 되겠죠?

## 문제 상황

단건 쿠폰을 조회할 때, 쿠폰과 연관된 예약 / 만남 정보를 함께 조회해야 했습니다. 하지만 `Coupon -> Meeting`, `Coupon -> Reservation`형태의 의존이 존재하지 않아 외부 패키지로의 접근이 불가피했는데요. 따라서 최초에는 아래와 같이 코드를 작성하였습니다.

```java
// coupon.application
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class CouponQueryService {

    private final CouponQueryRepository couponQueryRepository;
    private final CouponClient couponClient;
  
   //... 로직
}

// coupon.infrastructure
@Transactional(readOnly = true)
@Component
@RequiredArgsConstructor
public class CouponClient {

    private final ReservationRepository reservationRepository;
    private final MeetingRepository meetingRepository;

  // Response는 각 Reservation, Meeting 패키지에 존재한다.
    public ReservationResponse getReservationResponse(final Long couponId) {
        return ReservationResponse.of(
                reservationRepository.findTopByCouponIdAndReservationStatus(couponId, ReservationStatus.WAITING)
                        .orElseThrow(() -> new InvalidReservationException(ErrorType.NOT_FOUND_RESERVATION)));
    }

    public MeetingResponse getMeetingResponse(final Long couponId) {
        return MeetingResponse.of(
                meetingRepository.findTopByCouponIdAndMeetingStatus(couponId, MeetingStatus.ON_PROGRESS)
                        .orElseThrow(() -> new InvalidMeetingException(ErrorType.NOT_FOUND_MEETING)));
    }
}
```

상위 클래스인 `CouponQueryService` 에서 `CouponClient`를 사용하고 있습니다. 다른 패키지에 대한 직접 의존을 `CouponClient`라는 클래스로 만들어 막고 있지만 문제가 몇가지 있었습니다. 제가 생각한 문제는 다음과 같습니다.

1. 도메인 단위로 완벽히 분리된 시스템이 됐을 때, Repository에 대한 참조는 Http 요청으로 변경된다. 이때, `CouponClient`의 변경 내성이 급격히 낮아진다.
2. 도메인 단위로 완벽히 분리된 시스템이 됐을 때, 다른 패키지에 존재하는 각 `Response`들을 새로운 Response로 변경해야 한다.
3. 외부 시스템과 소통하는 `CouponClient`는 사실상 두 개의 다른 시스템을 참조한다. 따라서 `Meeting`의 변경, `Response`의 변경 모두 CouponClient에 영향을 준다.
4. `application` 계층에서 `infrastructure` 계층을 직접 참조한다. (안정적인 레이어드 아키텍처는 바로 상위 레이어만 하위 레이어를 참조할 수 있음)

위 문제점을 해결하기 위해 저는 `Separated Interface` 패턴을 도입했고, 부가적인 수정을 진행했습니다. 

## 1. CouponClient 인스턴스 변수를 인터페이스로 분리하라

외부 시스템을 사용하는 `CouponClient` 입장에서는 데이터를 **DB에서 값을 가져오든, 웹 요청으로 가져오든, 메모리에서 가져오든** 전혀 상관이 없습니다. `Coupon` 응답을 채우기 위한 값이 필요할 뿐이죠. 그말인즉슨, 어떤 요청으로 데이터를 받아오든 그 변경점이 `CouponClient`까지 넘어오면 안된다는 의미입니다.

따라서 저는 다음과 같이 `CouponClient`와 동일한 패키지에 인터페이스를 만들고 **다른 패키지와의 통합**이라는 의미의 `integrate` 패키지를 `infrastructure` 하위 패키지에 두어 두 패키지, 클래스간의 결합도를 크게 감소시킬 수 있었습니다.

```java
public interface ReservationProvider {

    ReservationResponse getWaitingReservation(Long couponId);
}

@Component
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class JpaReservationProvider implements ReservationProvider {

    private final ReservationRepository reservationRepository;

    @Override
    public ReservationResponse getWaitingReservation(final Long couponId) {
        return ReservationResponse.of(
                reservationRepository.findTopByCouponIdAndReservationStatus(couponId, ReservationStatus.WAITING)
                        .orElseThrow(() -> new InvalidReservationException(ErrorType.NOT_FOUND_RESERVATION)));
    }
}

// Meeting에 대한 참조도 같은 방식으로 변경
```

이렇게 변경하고 나니, 이후에 도메인이 서로 다른 시스템으로 완벽하게 분리되어 웹 요청으로 변경되더라도 그 변경점이 `CouponClient`로 까지 전파되지 않게 됐습니다.

## 2. Response를 이동하자

하지만 여전히 현재 패키지에서 다른 패키지의 `Response`를 참조하였는데요. 이 부분에는 여러가지 해결책(common 패키지, Mapper 등 )이 있겠지만 각 패키지에서는  `MeetingResponse`, `ReservationResponse`를 사용하지 않고 다른 Response를 쓰고 있었기 때문에 간단하게 `Coupon` 패키지의 `integrate` 패키지에 해당 응답값을 이동할 수 있었습니다.

![image-20220810125953543](https://user-images.githubusercontent.com/87312401/183817702-298021af-8b20-48e6-9c31-cc1ed9437f26.png)


이렇게 사용 중인 패키지로 클래스를 이동하니 독립된 시스템으로 배포될 때, 어떤 응답값이 오든 우리가 출력하고자 하는 `Response`로 매핑만 시켜주면 되니 분리에 대한 리팩터링 내성이 강해졌다고 생각합니다.

## 3. 단일 책임의 원칙을 지키자

현재 `CouponClient`는 서로 다른 두개의 시스템을 참조하여 값을 가져오는 역할을 하고 있습니다. 외부 api와 연결하는 역할을 하는 것은 좋지만 둘 중 하나의 응답에 대해 변경이 생겨도 항상 `CouponClient`가 변경된다는 단점이 있죠. 또, 단일 책임을 지키지 않은 것에 대한 몇 가지 단점들이 있을 것입니다.

게다가 현재 `CouponClient`는 아래와 같이 작성됐는데요.

```java
@Transactional(readOnly = true)
@Component
@RequiredArgsConstructor
public class CouponClient {

    private final ReservationProvider reservationProvider;
    private final MeetingProvider meetingProvider;

    public ReservationResponse getReservationResponse(final Long couponId) {
        return reservationProvider.getWaitingReservation(couponId);
    }

    public MeetingResponse getMeetingResponse(final Long couponId) {
        return meetingProvider.getOnProgressMeeting(couponId);
    }
}
```

`Provider` 인터페이스를 한 번 묶는 형태로만 사용하고 있습니다. 둘의 책임을 분리한다면, 굳이 `CouponClient`로 감싸줄 이유가 없어졌습니다. 

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class CouponQueryService {

    private final CouponQueryRepository couponQueryRepository;
    private final ReservationProvider reservationProvider;
    private final MeetingProvider meetingProvider;
		
    // ... 로직
}
```

여기에서 직접 인터페이스를 참조하는 형태로 사용할 수 있습니다. 덕분에 책임도 분리하였고 클래스도 간소화되었습니다. 그리고 마지막 문제점인 `application` 레이어에서 `infrastructure`를 참조하던 문제 또한 해결할 수 있습니다. `Provider` 인터페이스를 `domain` 레이어로 올리고 각각 `infrastructure`의 구현체를 참조하도록 변경하면 됩니다.

## 정리

결국 정리하자면 아래와 같습니다.

1. 변경점의 전파를 최대한 막자.
2. 인터페이스를 적극 활용하자.
3. 패키지 간 의존을 분리하는 것으로 변경점의 전파를 막을 수도 있다.

** 소스 코드는 [링크](https://github.com/woowacourse-teams/2022-thankoo)에서 확인할 수 있습니다.

긴 글 읽어주셔서 감사합니다~!
