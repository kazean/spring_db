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

