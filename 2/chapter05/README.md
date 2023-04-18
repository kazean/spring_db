# [이론] 5. 데이터 접근 기술 - JPA
## 1.JPA 시작
JPA 방대하고, 학습 분량이 많다, 그러나 매우 큰 생산성, SQL도 JPA가 대신 작성하고 처리해준다
> JPA를 더욱 편리하게 사용하기 위해 스프링 데이터 JPA와 Querydsl이라는 기술 사용
## 2.ORM 개념1 - SQL 중심적인 개발의 문제점 
## 3.ORM 개념2 - JPA 소개
## 4.JPA 설정
`spring-boot-starter-data-jpa` 라이브러리를 사용하면 JPA와 스프링 데이터 JPA를 스프링 부트와 통합하고, 설정도 아주 간단히 할 수 있다
- build.gradle
> spring-boot-starter-data-jpa는 spring-boot-starter-jdbc도 함께 포함(의존)한다.
>> hibernate-core: JPA 구현체  
jakarta-persistence-api: JPA 인터페이스  
spring-data-jpa: 스프링 데이터 JPA 라이브러리
- application.properties
> org.hibernateSQL=DEBUG: 하이버네이트가 생성하고 실행하는 SQL 확인  
org.hibernate.type.descriptor.sql.BasicBinder=TRACE: SQL에 바인딩 되는 파라미터 확인  
!spring.jpa.show-sql=true: System.out을 통해 출력 권장X

## 5.JPA 적용1 - 개발
- Item - `@Entity`
> @Entity: JPA가 사용하는 객체  
@Id: 테이블의 PK와 해당 필드를 매핑  
@GeneratedValue(strategy = GenerationType.IDENTITY): PK 생성 값을 데이터베이스에서 생성하는 IDENTITY 방식을 사용한다.  
@Column: 객체의 필드를 테이블 컬럼과 매핑(카멜, 언더스코어 자동 변환)  
JPA는 기본 생성자가 필수

- repository.jpa.JpaItemRepositoryV1
> private final EntityManager em: 스프링을 통해 엔티티 매니저라는 것을 주입, JPA의 모든 동작은 `EntityManager`를 통해 이루어진다  
`@Transactional`: JPA의 모든 데이터 변경(등록, 수정, 삭제)은 트랜잭션 안에서 이루어진다.  
em.persist(item), em.find(Item.class, id), em.createQuery(jpql, Item.class)
- cf
> JPA를 설정하려면 EntityManagerFactory, JPA 트랜잭션 매니저(JpaTransactionManager), 데이터소스 등 다양한 설정
>> 스프링 부트 자동화, JpaBaseConfiguration
- JpaConfig, ItemServiceApplication - 변경

## 6.JPA 적용2 - 리포지토리 분석
- save()
> `em.persist(item)`: JPA 객체를 테이블에 저장, PK키 생성 전략을 IDENTITY로 사용했기 때문에 JPA가 쿼리 만들어서 실행, Item 객체의 id 필드에 저장
- update()
> `em.update() 같은 메서드 전혀 호출하지 않았다.` JPA는 트랜잭션이 커밋되는 시점에 변경된 엔티티 객체가 있는지 확인 후 변경되었을 경우 update `영속성 컨텍스트`
- findById() - 단권 조회
> `em.find(Item.class, id)` JPA에서 PK를 기준으로 조회할 때는 find() 사용
- findAll() - 목록 조회
```
String jqpl = "select i from Item i";
TypedQuery<Item> query = em.createQuery(jpql, Item.class) ;
return query.getResultList();
 ```
- JPQL(Java Persistence Query Language) 
> JPA는 JPQL이라는 객체지향 쿼리 언어 제공, 주로 여러 데이터를 복잡한 조건으로 조회할 때 사용  
엔티티 객체와 속성의 대소문자는 구분해야 한다.
- JPQL 파라미터
> where price <: maxPrice
- !동적 쿼리 문제
> JPA 동적 쿼리 문제 > Querydsl이라는 기술 사용

## 7.JPA 적용3 - 예외 변환
JPA의 경우 예외가 발생하면 JPA 예외가 발생하게 된다.
> EntityManagers는 순수한 JPA 기술이고, 스프링과는 관계가 없다. > JPA 예외 발생  
JPA는 `PersistenceException`과 그 하위 예외를 발생 시킴 > + IllegalStateException, IllegalArgumentException  
JPA는 스프링 예외 추상화로 어떻게 변환? > `@Repository`
- @Repository의 기능
> 예외 변환 AOP의 적용 대상이 된다, 스프링 + JPA 경우: `JPA 예외 변환기(PersistenceExceptionTranslator)`를 등록
- cf 스프링 부트
> PersistenceExceptionTranslationPostProcessor 자동 등록 > @Repository를 AOP 프록시로 만드는 어드바이저가 등록된다.
- cf, EntityManagerFactoryUtils.convertJpaAccessExceptionIfPossible()