# bookdelivery
Lv.2 Intensive Coursework Group 3

<img src="https://user-images.githubusercontent.com/85722733/124438926-c0e43d80-ddb3-11eb-9d37-d89e8a7193eb.png"  width="50%" height="50%">

# 온라인 도서상점 (도서배송 서비스)

# Table of contents

- [조별과제 - 도서배송 서비스](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현)
    - [DDD 의 적용](#DDD-의-적용)
    - [동기식 호출과 Fallback 처리](#동기식-호출과-Fallback-처리)
    - [비동기식 호출과 Eventual Consistency](#비동기식-호출과-Eventual-Consistency)
    - [폴리글랏 퍼시스턴스/프로그래밍](#폴리글랏-퍼시스턴스-프로그래밍)
    - [API 게이트웨이](#API-게이트웨이)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

기능적 요구사항
1. 고객이 도서를 선택하여 주문(Order)한다
2. 고객이 결제(Pay)한다
3. 결제가 완료되면 주문 내역이 도서상점에 전달된다(Ordermanagement)
4. 상점주인이 주문을 접수하고 도서를 포장한다
5. 도서 포장이 완료되면 상점소속배달기사가 배송(Delivery)을 시작한다.
6. 고객이 주문을 취소할 수 있다
7. 주문이 취소되면 배송 및 결제가 취소된다
8. 고객이 주문상태를 중간중간 조회한다
9. 주문/배송상태가 바뀔 때마다 고객이 마이페이지에서 상태를 확인할 수 있다

비기능적 요구사항
1. 트랜잭션
  1. 결제가 완료되어야만 주문이 완료된다 (결제가 되지 않은 주문건은 아예 거래가 성립되지 않아야 한다 Sync 호출)
2. 장애격리
  1. 주문관리(Ordermanagement) 기능이 수행되지 않더라도 주문(Order)은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
  2. 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도한다 Circuit breaker, fallback
3. 성능
  1. 고객이 마이페이지에서 배송상태를 확인할 수 있어야 한다 CQRS


# 체크포인트

- 분석 설계


  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계


## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/487999/79684144-2a893200-826a-11ea-9a01-79927d3a0107.png)

## TO-BE 조직 (Vertically-Aligned)
  <img src="https://user-images.githubusercontent.com/85722733/124564081-a9708780-de7b-11eb-93aa-42c819be9059.png"  width="80%" height="80%">


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/2WHhr58frxP61Pm7rSPRUu6bGAY2/mine/348fa7c90636e01a5525272b163ef307


### 이벤트 도출
<img src="https://user-images.githubusercontent.com/85722733/124441029-39e49480-ddb6-11eb-8310-132caa4c887e.png"  width="80%" height="80%">

### 부적격 이벤트 탈락
<img src="https://user-images.githubusercontent.com/85722733/124441079-48cb4700-ddb6-11eb-8d12-57845e061f62.png"  width="80%" height="80%">

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
        - '주문내역이 상점에 전달됨' 및 '주문상태 업데이트됨'은 이벤트에 의한 반응에 가까우므로 이벤트에서 제외
        - '마이페이지에서 조회됨'은 발생한 사실, 결과라고 보기 어려우므로 이벤트에서 제외

### 액터, 커맨드 부착하여 읽기 좋게
<img src="https://user-images.githubusercontent.com/85722733/124451688-9ba9fc00-ddc0-11eb-815e-0e0c6f685b69.png"  width="65%" height="65%">

### 어그리게잇으로 묶기
<img src="https://user-images.githubusercontent.com/85722733/124451712-a5336400-ddc0-11eb-9561-e47f8b28b205.png"  width="80%" height="80%">

    - 고객의 주문, 상점의 주문관리, 결제의 결제이력, 배송의 배송이력은 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들끼리 묶어줌

### 바운디드 컨텍스트로 묶기

<img src="https://user-images.githubusercontent.com/85722733/124451753-aebccc00-ddc0-11eb-91ca-6b6355106898.png"  width="80%" height="80%">

    - 도메인 서열 분리 
        - Core Domain:  order, ordermanagement : 없어서는 안될 핵심 서비스이며, 연견 Up-time SLA 수준을 99.999% 목표, 배포주기는 order의 경우 1주일 1회 미만, ordermanagement의 경우 1개월 1회 미만
        - Supporting Domain:  delivery : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함
        - General Domain:   pay : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

<img src="https://user-images.githubusercontent.com/85722733/124451790-b7150700-ddc0-11eb-9e95-4cac51bb165e.png"  width="80%" height="80%">

### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)

<img src="https://user-images.githubusercontent.com/85722733/124451818-bf6d4200-ddc0-11eb-816d-8e55df0fdabc.png"  width="80%" height="80%">

### 완성된 1차 모형

![MSAEz](https://user-images.githubusercontent.com/85722733/124453306-36efa100-ddc2-11eb-9620-d07221ed7e78.png)

    - View Model 추가

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

<img src="https://user-images.githubusercontent.com/85722733/124564387-f8b6b800-de7b-11eb-8311-b9928bc13374.png"  width="80%" height="80%">

    - 고객이 도서를 선택하여 주문한다 (ok)
    - 고객이 결제한다 (ok)
    - 결제가 완료되면 주문 내역이 도서상점에 전달된다 (ok)
    - 상점주인이 주문을 접수하고 도서를 포장한다 (ok)
    - 도서 포장이 완료되면 상점소속배달기사가 배송을 시작한다 (ok)
    

<img src="https://user-images.githubusercontent.com/85722733/124564426-0409e380-de7c-11eb-8689-523340b2adf2.png"  width="80%" height="80%">
 
    - 고객이 주문을 취소할 수 있다 (ok)
    - 주문이 취소되면 배송 및 결제가 취소된다 (ok)
    - 고객이 주문상태를 중간중간 조회한다 (ok)
    - 주문/배송상태가 바뀔 때마다 고객이 마이페이지에서 상태를 확인할 수 있다 (ok)


### 비기능 요구사항에 대한 검증
<img src="https://user-images.githubusercontent.com/85722733/124566367-f190a980-de7d-11eb-9a9d-ba86558a095f.png"  width="80%" height="80%">

    - 마이크로서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
        - 고객 주문시 결제처리:  결제가 완료되지 않은 주문은 절대 받지 않는다는 경영자의 오랜 신념(?)에 따라, ACID 트랜잭션 적용. 주문완료시 결제처리에 대해서는 Request-Response 방식 처리
        - 결제 완료시 점주연결 및 배송처리:  payment 에서 ordermanagement 마이크로서비스로 주문요청이 전달되는 과정에 있어서 ordermanagement 마이크로서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
        - 나머지 모든 inter-microservice 트랜잭션: 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.




## 헥사고날 아키텍처 다이어그램 도출
    
![헥사고날아키텍쳐](https://user-images.githubusercontent.com/85722733/124441441-a364a300-ddb6-11eb-9307-8e4e1d324e29.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 Pub/Sub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐

# 구현 

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 바운더리 컨텍스트 별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd order
mvn spring-boot:run

cd payment
mvn spring-boot:run 

cd ordermanagement
mvn spring-boot:run  

cd delivery
mvn spring-boot:run 
```

## DDD 의 적용

- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가? 

각 서비스 내에 도출된 핵심 Aggregate Root 객체를 Entity로 선언하였다. (주문(order), 결제(payment), 주문관리(ordermgmt), 배송(delivery))

주문관리 Entity (Ordermgmt.java)
```
@Entity
@Table(name="Ordermgmt_table")
public class Ordermgmt {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long orderMgmtId;
    private Long orderId;
    private Long itemId;
    private String itemName;
    private Integer qty;
    private String customerName;
    private String deliveryAddress;
    private String deliveryPhoneNumber;
    private String orderStatus;

    @PostPersist
    public void onPostPersist(){
        OrderTaken orderTaken = new OrderTaken();
        BeanUtils.copyProperties(this, orderTaken);
        orderTaken.publishAfterCommit();
    }

    @PostUpdate
    public void onPostUpdate(){
        CancelOrderTaken cancelOrderTaken = new CancelOrderTaken();
        BeanUtils.copyProperties(this, cancelOrderTaken);
        cancelOrderTaken.publishAfterCommit();
    }

    public Long getOrderMgmtId() {
        return orderMgmtId;
    }

    public void setOrderMgmtId(Long orderMgmtId) {
        this.orderMgmtId = orderMgmtId;
    }

    public Long getOrderId() {
        return orderId;
    }

    public void setOrderId(Long orderId) {
        this.orderId = orderId;
    }

    public Long getItemId() {
        return itemId;
    }

    public void setItemId(Long itemId) {
        this.itemId = itemId;
    }

    public String getItemName() {
        return itemName;
    }
    .... 생략
```

Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 하였고 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다 

OrdermgmtRepository.java
```
package bookdelivery;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="ordermgmts", path="ordermgmts")
public interface OrdermgmtRepository extends PagingAndSortingRepository<Ordermgmt, Long>{

}
```
- REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가? 
```
미구현
```

- 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?

가능한 현업에서 사용하는 언어(유비쿼터스 랭귀지)를 모델링 및 구현 시 그대로 사용하려고 노력하였다.

- 적용 후 Rest API의 테스트

주문 결제 후 ordermgmts 주문 접수하기 POST
```
http localhost:8082/ordermgmts orderId=1 itemId=1 itemName="ITbook" qty=1 customerName="HanYongSun" deliveryAddress="kyungkido sungnamsi" deliveryPhoneNumber="01012341234" orderStatus="order"
```
![image](https://user-images.githubusercontent.com/78421066/124939757-5b5ab000-e044-11eb-808b-2f610e6a6677.png)

ordermgmts 주문 취소하기 PATCH 
```
http PATCH localhost:8082/ordermgmts/1 orderStatus="cancel"
```
![image](https://user-images.githubusercontent.com/78421066/124940062-9b219780-e044-11eb-92d5-579178b767bd.png)


## 동기식 호출과 Fallback 처리 
(Request-Response 방식의 서비스 중심 아키텍처 구현)

- 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)

요구사항대로 주문이 들어와야지만 결제 서비스를 호출할 수 있도록 주문 시 결제 처리를 동기식으로 호출하도록 한다. 

Order.java Entity Class에 @PostPersist로 주문 생성 직후 결제를 호출하도록 처리하였다
```
@PostPersist
    public void onPostPersist(){
        OrderPlaced orderPlaced = new OrderPlaced();
        BeanUtils.copyProperties(this, orderPlaced);
        orderPlaced.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        bookdelivery.external.Payment payment = new bookdelivery.external.Payment();
        //mappings goes here
        //add
        payment.setOrderId(this.getOrderId());
        payment.setItemPrice(this.getItemPrice());
        payment.setItemName(this.getItemName());
        payment.setQty(this.getQty());
        payment.setCustomerName(this.getCustomerName());
        payment.setDeliveryAddress(this.getDeliveryAddress());
        payment.setDeliveryPhoneNumber(this.getDeliveryPhoneNumber());
        OrderApplication.applicationContext.getBean(bookdelivery.external.PaymentService.class)
            .pay(payment);
    }
```
동기식 호출은 PaymentService 클래스를 두어 FeignClient 를 이용하여 호출하도록 하였다.

PaymentService.java

```
package bookdelivery.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="payment", url="http://localhost:8084", fallback = PaymentServiceFallback.class)
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void pay(@RequestBody Payment payment);

}
```
동기식 호출로 인하여, 결제 서비스에 장애 발생 시(서비스 다운) 주문 서비스에도 장애가 전파된다는 것을 확인
```
Order 서비스 구동 & Payment 서비스 다운 되어 있는 상태에서는 주문 생성 시 오류 발생

C:\workspace\bookdelivery>http POST localhost:8088/orders customerId=9005 customerName="Cho" itemId=4340 itemName="ABC" qty=2 itemPrice=1000 deliveryAddress="GwaCheon" deliveryPhoneNumber="01011112222" orderStatus="orderPlaced"

HTTP/1.1 500
Connection: close
Content-Type: application/json;charset=UTF-8
Date: Sun, 11 Jul 2021 14:44:14 GMT
Transfer-Encoding: chunked

{
    "error": "Internal Server Error",
    "message": "Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction",
    "path": "/orders",
    "status": 500,
    "timestamp": "2021-07-11T14:44:14.537+0000"
}

--> Payment 서비스 구동하여 주문 재생성 시 정상적으로 생성됨
C:\workspace\bookdelivery\payment>mvn spring-boot:run

C:\workspace\bookdelivery>http POST localhost:8088/orders customerId=9005 customerName="Cho" itemId=4340 itemName="ABC" qty=2 itemPrice=1000 deliveryAddress="GwaCheon" deliveryPhoneNumber="01011112222" orderStatus="orderPlaced"

HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Sun, 11 Jul 2021 14:50:14 GMT
Location: http://localhost:8081/orders/1
transfer-encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "customerId": 9005,
    "customerName": "Cho",
    "deliveryAddress": "GwaCheon",
    "deliveryPhoneNumber": "01011112222",
    "itemId": 4340,
    "itemName": "ABC",
    "itemPrice": 1000,
    "orderStatus": "orderPlaced",
    "qty": 2
}
```

- 서킷브레이커를 통하여 장애를 격리시킬 수 있는가?

주문-결제 Req-Res구조에서 FeignClient 및 Spring Hystrix 를 사용하여 Fallback 기능을 구현하였다

Order 서비스의 application.yml 파일에 feign.hystrix.enabled: true 로 활성화시킨다

```
feign:
  hystrix:
    enabled: true
```
PaymentService 에 feignClient fallback 옵션을 추가하였고 이를 위해 PaymentServiceFallback 클래스를 추가하였다

PaymentService.java
```
package bookdelivery.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="payment", url="http://localhost:8084", fallback = PaymentServiceFallback.class)
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void pay(@RequestBody Payment payment);

}
```
PaymentServiceFallback.java
```
package bookdelivery.external;

import org.springframework.stereotype.Component;

@Component
public class PaymentServiceFallback implements PaymentService{

  @Override
  public void pay(Payment payment) {
    System.out.println("Circuit breaker has been opened. Fallback returned instead.");
  }

}

```
fallback 기능 없이 payment 서비스를 중지하고 주문 생성 시에는 오류가 발생했으나, 

위와 같이 fallback 기능 활성화 후에는 payment서비스가 동작하지 않더라도 주문 생성 시에 오류가 발생하지 않는다

```
C:\workspace\bookdelivery> http POST localhost:8088/orders customerId=7777 customerName="HeidiCho" itemId=4340 itemName="ABC" qty=2 itemPrice=1000 deliveryAddress="GwaCheon" deliveryPhoneNumber="01011112222" orderStatus="orderPlaced"
HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Sun, 11 Jul 2021 15:54:23 GMT
Location: http://localhost:8081/orders/3
transfer-encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/3"
        },
        "self": {
            "href": "http://localhost:8081/orders/3"
        }
    },
    "customerId": 7777,
    "customerName": "HeidiCho",
    "deliveryAddress": "GwaCheon",
    "deliveryPhoneNumber": "01011112222",
    "itemId": 4340,
    "itemName": "ABC",
    "itemPrice": 1000,
    "orderStatus": "orderPlaced",
    "qty": 2
}
```
```
Hibernate:
    insert
    into
        order_table
        (customer_id, customer_name, delivery_address, delivery_phone_number, item_id, item_name, item_price, order_status, qty, order_id)
    values
        (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
2021-07-12 00:54:23.262 TRACE 12760 --- [nio-8081-exec-4] o.h.type.descriptor.sql.BasicBinder      : binding parameter [1] as [BIGINT] - [7777]
2021-07-12 00:54:23.262 TRACE 12760 --- [nio-8081-exec-4] o.h.type.descriptor.sql.BasicBinder      : binding parameter [2] as [VARCHAR] - [HeidiCho]
2021-07-12 00:54:23.263 TRACE 12760 --- [nio-8081-exec-4] o.h.type.descriptor.sql.BasicBinder      : binding parameter [3] as [VARCHAR] - [GwaCheon]
2021-07-12 00:54:23.263 TRACE 12760 --- [nio-8081-exec-4] o.h.type.descriptor.sql.BasicBinder      : binding parameter [4] as [VARCHAR] - [01011112222]
2021-07-12 00:54:23.263 TRACE 12760 --- [nio-8081-exec-4] o.h.type.descriptor.sql.BasicBinder      : binding parameter [5] as [BIGINT] - [4340]
2021-07-12 00:54:23.264 TRACE 12760 --- [nio-8081-exec-4] o.h.type.descriptor.sql.BasicBinder      : binding parameter [6] as [VARCHAR] - [ABC]
2021-07-12 00:54:23.264 TRACE 12760 --- [nio-8081-exec-4] o.h.type.descriptor.sql.BasicBinder      : binding parameter [7] as [INTEGER] - [1000]
2021-07-12 00:54:23.265 TRACE 12760 --- [nio-8081-exec-4] o.h.type.descriptor.sql.BasicBinder      : binding parameter [8] as [VARCHAR] - [orderPlaced]
2021-07-12 00:54:23.265 TRACE 12760 --- [nio-8081-exec-4] o.h.type.descriptor.sql.BasicBinder      : binding parameter [9] as [INTEGER] - [2]
2021-07-12 00:54:23.265 TRACE 12760 --- [nio-8081-exec-4] o.h.type.descriptor.sql.BasicBinder      : binding parameter [10] as [BIGINT] - [3]
2021-07-12 00:54:23.268 DEBUG 12760 --- [strix-payment-2] o.s.c.openfeign.support.SpringEncoder    : Writing [bookdelivery.external.Payment@2d962b78] using [org.springframework.http.converter.json.MappingJackson2HttpMessageConverter@2e4166b7]
Circuit breaker has been opened. Fallback returned instead.
```
위와 같이 fallack 옵션이 동작하여 "Circuit breaker has been opened. Fallback returned instead." 로그가 보여진다


## 비동기식 호출과 Eventual Consistency 
(이벤트 드리븐 아키텍처)

- 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?

- Correlation-key: 각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?

카프카를 이용하여 주문완료 시 결제 처리를 제외한 나머지 모든 마이크로서비스 트랜잭션은 Pub/Sub 관계로 구현하였다. 

아래는 주문취소 이벤트(OrderCanceled)를 카프카를 통해 주문관리(ordermanagement) 서비스에 연계받는 코드 내용이다. 

order 서비스에서는 고객이 주문 취소 시 PostUpdate로 OrderCanceled 이벤트를 발생시키고,
```
public class Order {
    @PostUpdate
      public void onPostUpdate(){
        OrderCanceled orderCanceled = new OrderCanceled();
        BeanUtils.copyProperties(this, orderCanceled);
        orderCanceled.publishAfterCommit();
    }
```

ordermanagement 서비스에서는 카프카 리스너를 통해 order의 OrderCanceled 이벤트를 수신받아서 폴리시(cancelOrder) 처리하였다. (getOrderId()를 호출하여 Correlation-key 연결)
```
@Service
public class PolicyHandler{
  @StreamListener(KafkaProcessor.INPUT)
    public void wheneverOrderCanceled_CancelOrder(@Payload OrderCanceled orderCanceled){

        if(!orderCanceled.validate()) return;

        System.out.println("\n\n##### listener CancelOrder : " + orderCanceled.toJson() + "\n\n");

        // 주문 취소시 상태 UPDATE 필요, Correlation-key 연결
        ordermgmtRepository.findByOrderId(orderCanceled.getOrderId()).ifPresent(ordermgmt->{
            ordermgmtRepository.save(ordermgmt);
        });
    }
```


- Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가?

배송(delievery)서비스의 포트 추가(기존:8083, 추가:8093)하여 2개의 노드로 배송서비스를 실행한다. bookdelivery topic의 partition은 1개이기 때문에 기존 8083 포트의 서비스만 partition이 할당된다.
![image](https://user-images.githubusercontent.com/78421066/125026479-a534ac00-e0bf-11eb-878c-0a4e6cf3c5d9.png)


주문관리서비스(ordermanagement)에서 이벤트가 발생하면 8083포트에 있는 delivery서비스에게만 이벤트 메세지가 수신되게 된다.
```
##### listener StartDelivery : {"eventType":"OrderTaken","timestamp":"20210709140205","orderMgmtId":6,"orderId":1,"
itemId":1,"itemName":"ITbook","qty":1,"customerName":"HanYongSun","deliveryAddress":"kyungkido sungnamsi","delivery
PhoneNumber":"01012341234","orderStatus":"order"}


Hibernate:
    call next value for hibernate_sequence
Hibernate:
    insert
    into
        delivery_table
        (customer_name, delivery_address, delivery_phone_number, order_id, order_status, delivery_id)
    values
        (?, ?, ?, ?, ?, ?)
```

8093포트의 delivery서비스의 경우 메세지를 수신받지 못한다.

```
변동사항 없음
```

8083 포트를 중지 시키면 8093포트의 delivery 서비스에서 partition을 할당 받는다
![image](https://user-images.githubusercontent.com/78421066/125026249-1fb0fc00-e0bf-11eb-9af2-d9888005c67a.png)

- 취소에 따른 보상 트랜잭션을 설계하였는가(Saga Pattern)
```
추가필요
```
- CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

주문/배송상태가 바뀔 때마다 고객이 마이페이지에서 상태를 확인할 수 있어야 한다는 요구사항에 따라 주문 서비스 내에 MyPage View를 모델링하였다

![mypage](https://user-images.githubusercontent.com/85722733/125193030-68b2ad00-e285-11eb-9261-4b2dbf5cfb91.png)

주문에 대한 결제완료(PayApproved) 시 orderId를 키값으로 MyPage 데이터도 생성되며 (요구사항으로 결제가 완료된 건에 대해서만 주문으로 인정하므로)

"결제완료(주문완료), 주문접수, 배송시작, 결제취소(주문취소)"의 이벤트에 따라 주문상태가 업데이트되도록 모델링하였다

MyPage View 의 속성값

![속성값](https://user-images.githubusercontent.com/85722733/125192987-4f116580-e285-11eb-985b-9355ab17385a.png)

MSAEz 모델링 도구 내 View CQRS 설정 샘플

![CQRS설정](https://user-images.githubusercontent.com/85722733/125193008-5afd2780-e285-11eb-8b54-67078edbffaf.png)

자동생성된 소스는 아래와 같다

MyPage CQRS처리를 위해 주문, 결제, 주문관리, 배송 서비스와 별개로 조회를 위한 MyPage_table 테이블이 생성된다

MyPage.java : 엔티티 클래스
```
package bookdelivery;

import javax.persistence.*;
import java.util.List;

@Entity
@Table(name="MyPage_table")
public class MyPage {

        @Id
        @GeneratedValue(strategy=GenerationType.AUTO)
        private Long orderId;
        private String customerName;
        private String itemName;
        private Integer qty;
        private Integer itemPrice;
        private String orderStatus;


        public Long getOrderId() {
            return orderId;
        }

        public void setOrderId(Long orderId) {
            this.orderId = orderId;
        }
        public String getCustomerName() {
            return customerName;
        }

        public void setCustomerName(String customerName) {
            this.customerName = customerName;
        }
        public String getItemName() {
            return itemName;
        }

        public void setItemName(String itemName) {
            this.itemName = itemName;
        }
        public Integer getQty() {
            return qty;
        }

        public void setQty(Integer qty) {
            this.qty = qty;
        }
        public Integer getItemPrice() {
            return itemPrice;
        }

        public void setItemPrice(Integer itemPrice) {
            this.itemPrice = itemPrice;
        }
        public String getOrderStatus() {
            return orderStatus;
        }

        public void setOrderStatus(String orderStatus) {
            this.orderStatus = orderStatus;
        }

}
```
MyPageRepository.java : 퍼시스턴스
```
package bookdelivery;

import org.springframework.data.repository.CrudRepository;
import org.springframework.data.repository.query.Param;

import java.util.List;

public interface MyPageRepository extends CrudRepository<MyPage, Long> {

}
```
MyPageViewHandler.java : 아래와 같이 결제완료를 통한 MyPage 주문 데이터 생성 및 주문상태 변경에 대한 이벤트 수신 처리부가 있다

주문에 대한 결제완료 시 이벤트
```
@StreamListener(KafkaProcessor.INPUT)
    public void whenPayApproved_then_CREATE_1 (@Payload PayApproved payApproved) {
        try {

            if (!payApproved.validate()) return;

            // view 객체 생성
            MyPage myPage = new MyPage();
            // view 객체에 이벤트의 Value 를 set 함
            myPage.setOrderId(payApproved.getOrderId());
            myPage.setCustomerName(payApproved.getCustomerName());
            myPage.setItemName(payApproved.getItemName());
            myPage.setQty(payApproved.getQty());
            myPage.setItemPrice(payApproved.getItemPrice());
            myPage.setOrderStatus(payApproved.getOrderStatus());
            // view 레파지 토리에 save
            myPageRepository.save(myPage);
        
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```
주문상태 업데이트 이벤트
```
@StreamListener(KafkaProcessor.INPUT)
    public void whenOrderTaken_then_UPDATE_1(@Payload OrderTaken orderTaken) {
        try {
            if (!orderTaken.validate()) return;
                // view 객체 조회
            Optional<MyPage> myPageOptional = myPageRepository.findById(orderTaken.getOrderId());
            if( myPageOptional.isPresent()) {
                MyPage myPage = myPageOptional.get();
                // view 객체에 이벤트의 eventDirectValue 를 set 함
                    myPage.setOrderStatus(orderTaken.getOrderStatus());
                // view 레파지 토리에 save
                myPageRepository.save(myPage);
            }
            
        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenDeliveryStarted_then_UPDATE_2(@Payload DeliveryStarted deliveryStarted) {
        try {
            if (!deliveryStarted.validate()) return;
                // view 객체 조회
            Optional<MyPage> myPageOptional = myPageRepository.findById(deliveryStarted.getOrderId());
            if( myPageOptional.isPresent()) {
                MyPage myPage = myPageOptional.get();
                // view 객체에 이벤트의 eventDirectValue 를 set 함
                    myPage.setOrderStatus(deliveryStarted.getOrderStatus());
                // view 레파지 토리에 save
                myPageRepository.save(myPage);
            }
            
        }catch (Exception e){
            e.printStackTrace();
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void whenPayCanceled_then_UPDATE_3(@Payload PayCanceled payCanceled) {
        try {
            if (!payCanceled.validate()) return;
                // view 객체 조회
            Optional<MyPage> myPageOptional = myPageRepository.findById(payCanceled.getOrderId());
            if( myPageOptional.isPresent()) {
                MyPage myPage = myPageOptional.get();
                // view 객체에 이벤트의 eventDirectValue 를 set 함
                    myPage.setOrderStatus(payCanceled.getOrderStatus());
                // view 레파지 토리에 save
                myPageRepository.save(myPage);
            }
            
        }catch (Exception e){
            e.printStackTrace();
        }
    }
```

CQRS 테스트

주문에 대한 결제완료 시 주문 정상 등록됨을 확인

![1_payment발생](https://user-images.githubusercontent.com/85722733/125193044-7b2ce680-e285-11eb-9756-36c608bf30ab.png)

아래와 같이 MyPage에도 주문상태가 'payApproved:orderFinallyPlaced'로 정상 등록되어 조회됨을 확인

![4_결제완료시주문완료로mypage상태업데이트](https://user-images.githubusercontent.com/85722733/125193056-8bdd5c80-e285-11eb-8457-a9771b96aded.png)

점주가 주문 접수건 발생 시에는 배송시작 이벤트가 발행되어 MyPage에 해당 주문 건에 대한 주문상태가 'deliveryStarted' 상태로 변경되어 조회됨을 확인

![5_점주가주문접수하여ordermgmt생성](https://user-images.githubusercontent.com/85722733/125193070-9ef02c80-e285-11eb-9629-06928f76cf17.png)

![6_주문접수및배달시작이벤트발생](https://user-images.githubusercontent.com/85722733/125193423-52a5ec00-e287-11eb-8c2e-c388a747d017.png)

![7_배달시작시mypage상태업데이트](https://user-images.githubusercontent.com/85722733/125193080-a6afd100-e285-11eb-9116-15670b9a842c.png)

주문접수취소에 따른 결제취소완료 시 최종주문취소로 간주하여 주문상태가 'OrderFinallyCanceled'로 변경되며 MyPage에 해당 주문 건에 대한 주문상태가 'orderFinallyCanceled'로 동일하게 조회된다

![8_주문접수취소](https://user-images.githubusercontent.com/85722733/125193096-b29b9300-e285-11eb-9578-0adb198bc557.png)

![11_주문접수취소및결제취소및배달취소이벤트발생](https://user-images.githubusercontent.com/85722733/125193261-7fa5cf00-e286-11eb-8e69-69b00f2b5ec3.png)

![10_결제취소시mypage상태업데이트](https://user-images.githubusercontent.com/85722733/125193121-d52dac00-e285-11eb-9eea-e508c98b23bc.png)

- Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?

ordermanagement 서비스만 구동되고 delivery 서비스는 멈춰있는 상태이다. 주문관리에 이벤트가 발생하면 카프카 큐에 정상적으로 들어감을 확인할 수 있다.
```
주문관리 이벤트 생성
$ http localhost:8082/ordermgmts orderId=1 itemId=1 itemName="ITbook" qty=1 customerName="HanYongSun" deliveryAddress="kyungkido sungnamsi" deliveryPhoneNumber="01012341234" orderStatus="order"
HTTP/1.1 201
Content-Type: application/json;charset=UTF-8
Date: Thu, 08 Jul 2021 23:16:56 GMT
Location: http://localhost:8082/ordermgmts/1
Transfer-Encoding: chunked

{
    "_links": {
        "ordermgmt": {
            "href": "http://localhost:8082/ordermgmts/1"
        },
        "self": {
            "href": "http://localhost:8082/ordermgmts/1"
        }
    },
    "customerName": "HanYongSun",
    "deliveryAddress": "kyungkido sungnamsi",
    "deliveryPhoneNumber": "01012341234",
    "itemId": 1,
    "itemName": "ITbook",
    "orderId": 1,
    "orderStatus": "order",
    "qty": 1
}
```
카프카 Consumer 캡쳐
![image](https://user-images.githubusercontent.com/78421066/125002634-5d4a6080-e090-11eb-8e55-994bf1c64a33.png)


배송(delivery)서비스 실행 및 실행 후 카프카에 적재된 메세지 수신 확인
```
cd delivery
mvn spring-boot:run

##### listener StartDelivery : {"eventType":"OrderTaken","timestamp":"20210709081656","orderMgmtId":1,"orderId":1,"itemId":1,"itemName":"ITbook","qty":1,"customerName":"HanYongSun","deliveryAddress":"kyungkido sungnamsi","del
iveryPhoneNumber":"01012341234","orderStatus":"order"}


Hibernate:
    call next value for hibernate_sequence
Hibernate:
    insert
    into
        delivery_table
        (customer_name, delivery_address, delivery_phone_number, order_id, order_status, delivery_id)
    values
        (?, ?, ?, ?, ?, ?)
```
카프카 Consumer 캡쳐
![image](https://user-images.githubusercontent.com/78421066/125002840-ca5df600-e090-11eb-992c-ed72ee7cfca8.png)


## 폴리글랏 퍼시스턴스/프로그래밍
- 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
```
추가필요
```

- 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
```
미구현
```


## API 게이트웨이

- API GW를 통하여 마이크로 서비스들의 진입점을 통일할 수 있는가?

아래는 MSAEZ를 통해 자동 생성된 gateway 서비스의 application.yml이며, 마이크로서비스들의 진입점을 통일하여 URL Path에 따라서 마이크로서비스별 서로 다른 포트로 라우팅시키도록 설정되었다.

gateway 서비스의 application.yml 파일 

```
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://localhost:8081
          predicates:
            - Path=/orders/**, /myPages/**
        - id: ordermanagement
          uri: http://localhost:8082
          predicates:
            - Path=/ordermgmts/** 
        - id: delivery
          uri: http://localhost:8083
          predicates:
            - Path=/deliveries/** 
        - id: payment
          uri: http://localhost:8084
          predicates:
            - Path=/payments/** 
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

---
```

Gateway 포트인 8088을 통해서 주문을 생성시켜 8081 포트에서 서비스되고 있는 주문서비스(order)가 정상 동작함을 확인함

![GW2](https://user-images.githubusercontent.com/85722733/125040735-f13d1c00-e0d2-11eb-9a60-e2f1ba6a5e51.png)

- 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?

gateway(8088 포트)를 통한 orders, payments, ordermgmts, deliveries 경로 접근은 차단하도록 환경 설정을 하였다.
pathMatchers("/oauth/","/login/").permitAll() : /oauth/, /login/ 경로만 게이트웨이에서 접근이 가능하도록 하였다.
oauth2ResourceServer() : 인증서버를 이용, jwt() : jwt 방식 인증

```
   @Bean
    SecurityWebFilterChain springSecurityFilterChain(ServerHttpSecurity http) throws Exception {

        http
                .cors().and()
                .csrf().disable()
                .authorizeExchange()
                .pathMatchers("/oauth/**","/login/**").permitAll()
                .anyExchange().authenticated()
                .and()
                .oauth2ResourceServer()
                .jwt()
                ;

        return http.build();
    }
```

인증서버(OAuth)의 경우 해당 아이디(1@uengine.org)와 패스워드(1)로 접근한 사용자만 토큰값을 얻어서 접근 할 수 있도록 구현하였다.
```
		User user = new User();
		user.setUsername("1@uengine.org");
		user.setPassword(passwordEncoder.encode("1"));
		user.setNickName("유엔진");
		user.setAddress("서울시");
		user.setRole("USER_ADMIN");
		repository.save(user);
```

인증서버(OAuth)와 gateway서버(gateway-master)를 실행시킨다.
```
cd OAuth
mvn spring-boot:run

cd gateway-master
mvn spring-boot:run
```

gateway(8088포트)를 통해 ordermgmts로 접근을 하면 유효하지 않은 인증(401 Unauthorized) 나오게 된다.

```
$ http localhost:8088/ordermgmts
HTTP/1.1 401 Unauthorized
Cache-Control: no-cache, no-store, max-age=0, must-revalidate
Expires: 0
Pragma: no-cache
Referrer-Policy: no-referrer
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
WWW-Authenticate: Bearer
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 1 ; mode=block
content-length: 0
```

인증을 하기 위해서 토큰값을 갖고 온다.

```
http --form POST localhost:8090/oauth/token "Authorization: Basic dWVuZ2luZS1jbGllbnQ6dWVuZ2luZS1zZWNyZXQ=" grant_type=password username=1@uengine.org password=1
```
![image](https://user-images.githubusercontent.com/78421066/125151912-8ba96800-e184-11eb-8523-c816453bcd27.png)

해당 access_token 값을 가지고 다시 localhost:8088/ordermgmts에 접속하면 인증됨을 확인 할 수 있다.

```
http localhost:8088/ordermgmts "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhZGRyZXNzIjoi7ISc7Jq47IucIiwidXNlcl9uYW1lIjoiMUB1ZW5naW5lLm9yZyIsInNjb3BlIjpbInJlYWQiLCJ3cml0ZSIsInRydXN0Il0sIm5pY2tuYW1lIjoi7Jyg7JeU7KeEIiwiY29tcGFueSI6IlVlbmdpbmUiLCJleHAiOjE2MjU5Nzg0NjQsImF1dGhvcml0aWVzIjpbIlVTRVJfQURNSU4iXSwianRpIjoiZ1l6cEltL29RYytucC9iYVZacGZYazNIU3k0PSIsImNsaWVudF9pZCI6InVlbmdpbmUtY2xpZW50In0.Ic56B-RPB4voEPSnQ_IecmSwbgqg2x7FojMFohKvHzMnKzA_6yb72vFs-ay3T7DSyplD22bdHvE1yEYV8oTzAv47srcjS4YLMnM9BDVLartkltfaj-DkXuiNRDbvesIKp4tTv3gFEQ16deocvY9W5Dv-Hkhqk_Hy4SlR2LKdKD2Q5yHDM4kqsNesjPFnRydJqHLgv0l9LIF76VJI5woMFJ8H6mRGE8DKJOvOF2DwItc8MzqgwILQV4WYzw8yRy_CZjR2hDG1wsqqhi1YlQWfgySRrFsaXAYv08h_rMPzudpncNOXM1i9SZlXcX0-BI03GCO6RmLMmo-NonTkSk5JTg"
```
![image](https://user-images.githubusercontent.com/78421066/125152033-30c44080-e185-11eb-902e-b9151c180b8c.png)

# 운영
## CI/CD 설정
## 동기식 호출 / 서킷 브레이킹 / 장애격리
## 오토스케일 아웃
## 무정지 재배포

