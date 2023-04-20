# [이론] 8.데이터 접근 기술 - 활용 방안
## 1.스프링 데이터 JPA 예제와 트레이드 오프
- 클래스 의존 관계
- 런타임 객체 의존 관계
> ItemService > JpaItemRepositoryV2 > SpringDataJpaItemRepository
>> 중간에서 JpaItemRepositoryV2가 어댑터 역할, MemberService 유지
- 고민  
> 구조를 맞추기 위해서, 중간에 어댑터가 들어가면 전체 구조가 너무 복잡, 클래스도 많아진다.  
> 유지보수 관점 DI, OCP, 어댑터 코드와 실제 코드까지 함께 유지보수 해야하는 어려움

- 다른 선택  
ItemService 코드를 일부 고쳐서 직접 스프링 데이터 JPA를 사용하는 방법
- 클래스 의존 관계, 런타임 의존 관계
> ItemService > SpringDataJpaItemRepository
>> 트레이드 오프

## 2.실용적인 구조
Querydsl을 사용한 리포지토리 스프링 데이터 JPA를 사용하지 않는 아쉬움
> 스프링 데이터 JPA + QueryDsl  
> ItemSerice > ItemRepositoryV2(SpringData), ItemQueryRepositoryV2
- hello.itemservice.repository.v2.ItemRepositoryV2
> extends JpaRepository<Item, Long>
- hello.itemservice.repository.v2.ItemQueryRepository
- hello.itemservice.service.ItemServiceV2
```
private final ItemRepositortV2 itemRepositoryV2;
private fianl ItemQueryRepositoryV2 itemQueryRepositoryV2;
```
- V2Config, ItemServiceApplication - 변경
> 테스트, 애플리케이션

## 3.다양한 데이터 접근 기술 조합
비지니스 상황, 현재프로젝트 구성원의 역량에 따라 결정
- 트랜잭션 매니저 선택
> JpaTransactionManager(JPA), JdbcTransactionManager(JdbcTemplate, Mybatis)
- JpaTransactionManager의 다양한 지원
> JdbcTransactionManager가 제공하는 기능도 대부분 제공

- 주의점
> 한 트랜잭션안에 JPA와 JdbcTemplate같이 사용할 경우 JPA를 통한 데이터를 JdbcTemplate에서 읽지 못하는 경우
>> JPA 호출이 끝난 시점에 플러시라는 기능 사용하여 DB에 반영해주어야 한다.

## 4.정리
ItemServiceV2는 ItemRepositoryV2(DataJPA)도 참조하고 ItemQueryRepositoryV2도 직접 참조한다.
> 복잡함 없이 단순하게 개발할 수 있다.  
!리포지토리의 구현 기술이 변경되면 수 많은 코드를 변경해야하는 단점