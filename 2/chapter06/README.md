# [실습] 6.데이터 접근 기술 - 스프링 데이터 JPA
## 1.스프링 데이터 JPA 소개1 - 등장 이유
## 2.스프링 데이터 JPA 소개2 - 기능
## 3.스프링 데이터 JPA 주요 기능
`공통 인터페이스 기능`  
`쿼리 메소드 기능`
- `공통 인터페이스 기능`
> Repository > CrudRepository > PagingAndSortingRepository  
JPARepository
>> JpaRepository 인터페이스를 통해서 CRUD, 공통화 가능한 기능
- JpaRepository 사용법
> public interface ItemRepository extends JpaRepository<Member, Long>
- 스프링 데이터 JPA가 구현 클래스르 대신 생성

- `쿼리 메소드 기능`  
인터페이스에 메소드만 적어두면, 메소드 이름을 분석해서 쿼리를 자동으로 만들고 실행해주는 기능
- 스프링 데이터 JPA
> List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
- 스프링 데이터 JPA가 제공하는 쿼리 메소드 기능
> 조회: find...By, read...By: ... > 설명(description)  
COUNT: count...By: long  
EXISTS: exists...By: boolean  
삭제: delete...By, remove...By: long  
DISTINCT: findDistinct, findMemberDistinctBy  
LIMIT: findFirst3, findFirst, findTop, findTop3
- 쿼리 메소드 필터 조건
> https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#jpa.querymethods.query-creation  
https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.limitquery-result

- JPQL 직접 사용하기
```
@Query(select i from Item i where i.itemName like :itemName and i.price < :maxPrice)
List<Item> findItems(@Param("itemName") String itemName, @Param("price"));
```

## 4. 스프링 데이터 JPA 적용1
- build.gradle > spring-boot-starter-data-jpa
- 스프링 데이터 JPA 적용 - hello.itemservice.repository.jpa.SpringDataJpaItemRepository
```
public interface SprinDataJpaItemRepository extends JpaRepository<Item, Long> {
	List<Item> findByItemNameLike(String itemName);
	List<Item> findByPriceLessThanEqual(Integer price);
	List<Item> findByItemNameLikeAndPriceLessThanEqual(String itemNae, Integer price);
	// 쿼리 직접 실행
	@Query(..)
	~
}
```
> CRUD, 쿼리 메소드 기능, @Query  
!스프링 데이터 JPA는 동적 쿼리에 약하다

## 5.스프링 데이터 JPA 적용2
- JpaItemRepositoryV2
> private final SpringDataJpaItemRepository repository
- 의존관계와 구조
> ItemService는 ItemRepository에 의존하기 때문에 ItemServie에서 SpringDataJpaItemRepository를 그대로 사용할 수 없다.  
Service 코드 변경없이 DI 구현하려면?
- 클래스 의존 관계
- 런타임 객체 의존 관계
> ItemService > ItemRepositoryV2 > SpringDataJpa
- SpringDataJpaConfig, ItemServiceApplication
- 예외 변환
> 스프링 데이터 JPA도 스프링 예외 추상화를 지원한다. 스프링 데이터 JPA가 만들어주는 프록시에서 이미 예외 변환을 처리하기 때무넹, @Repository와 관계없이 예외가 변환된다.
- 주의! - 하이버네이트 버그
> 하이버네이트 5.6.6 ~ 5.6.7 like bugg
>> ext["hibernate.version"] = "5.6.5.Final"

## 6.정리
스프링 데이터 JPA는 실무에서 기본으로 선택하는 기술이다.