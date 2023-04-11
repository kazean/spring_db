# [실습] 1. JDBC 이해
## 1. 프로젝트 생성
- spring : 3.0, jdk17
- Gradle Project, Package: hello.jdbc, Group: hello, Artifact: jdbc
- Dependecies: spring-boot-starter-jdbc, lombok, h2

## 2. H2 데이터베이스 설정
- h2
```
h2 Version 2.1.214
username: sa
url: (최초)jdbc:h2:~/test, (이후) jdbc:h2:tcp://localhost/~/test
```
- Member table create sql
```
drop table member if exists cascade;
create table member (
                        member_id varchar(10),
                        money integer not null default 0,
                        primary key (member_id)
);
insert into member(member_id, money) values ('hi1',10000);
insert into member(member_id, money) values ('hi2',20000);
```

## 3. JDBC 이해
- jdbc 등장 이유
문제는 각각 데이터베이스 마다 커넥션을 연결하는 방법, SQL을 전달하는 방법, 그리고 결과를 응답 받는 방법 모두 다르다  
크게 2가지문제
> 1) 데이터베이스를 다른 종류의 데이터베이스로 변경시 코드변경  
2) 개발자가 각각의 데이터베이스마다 커넥션 연결, SQL 전달, 그리고 그 결과를 응답받는 방법을 새로 학습
- `JDBC 표준 인터페이스`
JDBC(Java Database Connectivity): 자바에서 데이터베이스에 접속할 수 있도록 하는 자바 API다
> `java.sql.Connection, Statement, ResultSet`  
자바는 이렇게 표준 인터페이스를 정의해두었다.  
그런데 인터페이스만 있다고해서 기능이 동작하지 않는다. DB 벤더에서 자신의 DB에 맞도록 구현해서 라이브러리로 제공
- 정리
JDBC 등장으로 2가지 문제가 해결  
1. 데이터베이스로 변경하면 코드 변경해야되는 문제
2. 개발자가 각각의 데이터베이스 마다 커넥션, SQL전달, 결과 응답 받는 방법을 새로 학습해야하는 문제
- cf, 표준화의 한계
JDBC 등장으로 ANSI SQL이라는 표준이 있긴 하지만 공통화의 한계  
ex) DB마다 페이징 기법이 다름  
> JPA를 사용하면 각각의 데이터베이스마다 다른 SQL을 정의해야하는 문제도 많은 부분 해결가능

## 4. JDBC와 최신 데이터 접근 기술
JDBC를 편리하게 사용하는 다양한 기술이 존재한다. 대표적으로 `SQL Mapper와 ORM기술`로 나뉜다
- SQL Mapper
> 장점
>> JDBC를 편리하게 사용하도록 도와준다.  
SQL 응답 결과를 객체로 편리하게 변환해준다.  
JDBC의 반복 코드를 제거해준다.
> 단점
>> 개발자가 SQL을 직접 작성해야한다.
> 대표기술
>> 스프링 JdbcTemplate, Mybatis
- ORM 기술
> ORM은 객체를 관계형 데이터베이스 테이블과 매핑해주는 기술이다.
> 대표기술
>> JPA, 하이버네이트, 이클립스 링크

## 5. 데이터베이스 연결
- hello.jdbc.connection.ConnectionConst
```
static final String URL, USERNAME, PASSWORD
```
- hello.jdbc.connection.DBConnectionUtil
```
static Connection getConnection() {
    DriverManager.getConection(URL, USERNAME, PASSWORD)
}
```
> 데이터베이스에 연결하려면 JDBC가 제공하는 `DrigetManager.getConnection(..)`사용
>> DBConnectionUtilTest
- JDBC DriverManager 연결 이해
java.sql.Connection 표준 커넥션 인터페이스를 정의 > org.h2.JdbcConenction  
JDBC가 제공하는 DriverManager는 라이브러리에 등록된 드라이브러 목록으 자동으로 인식한다, 이 드라이버들에게 순서대로 정보를 넘겨 커넥션을 할 수 있는지 확인

## 6. JDBC 개발 - 등록
- member table 만들기
```
drop table member if exists cascade;
create table member (
member_id varchar(10),
money integer not null default 0,
primary key (member_id)
);
```
- hello.jdbc.domain.Member
> memberId, money
- hello.jdbc.repository.MemberRepositoryV0 - 회원등록
```
public Member save(Member member) {
    ...
    con = getConnection();
    pstmt = con.preparedStatement(sql);
    pstmt.setString(1, member.getMemberId())
    pstmt.setString(2, member.getMoney())
    pstmt.executeUpdate()
    ...
}

private Connection getConnection() {
    ~
}
private void close(Connection con, Statement stmt, ResultSet rs) {
    ~
}
```
> 커넥션 획득: getConnection()
> save() - SQL 전달
>> pstmt.setString(1, member.getMemberId()): SQL의 첫번째 ?에 값을 지정한다.
> psmt.executeUpdate(): Statement를 통해 준비된 SQL을 커넥션을 통해 실제 데이터베이스에 전달한다.
```
int executeUpdate() throws SQLException;
```
> 리소스 정리: 하지 않게 되면 커넥션이 끊어지지않고 계속 유지되는 문제
- MemberRepositoryV0Test - 회원등록
> db: select * from member;

## 7. JDBC 개발 - 조회
- MemberRepositoryV0 - 회원조회추가
```
public Member findByid(String memberId) {
	String sql = "select * from member where member_id = ?";
	rs = pstmt.executeQuery();
	if(rs.next()) {
			Member member ~
			return member;
	} else {
			throw new NoSuchElementException("member not found memberId=" + memberId);
	}
}
```
> rs = pstmt.executeQuery(): 데이터 변경할 때는 executeUpdate()를 사용하지만, 데이터 조회시 executeQuery()
- executeQuery()
```
ResultSet executeQuery() throws SQLException;
```
- ResultSet
> 보통 select 쿼리의 결과가 순서대로 들어간다.  
내부에 있는 커서를 이동해서 다음 데이터를 조회  
rs.next(): 이것을 호출하면 커서가 다음으로 이동, 참고로 최초의 커서는 데이터를 가르키고 있지 않기 때문에 rs.next()를 최초 한번은 호출해야 데이터를 조회할 수 있다.  
rs.getString("member_id"), rs.getInt("money)"
- MemberRepositoryV0Test - 회원 조회 추가

## 8. JDBC 개발 - 수정, 삭제
- MemberRepositoryV0 - 회원 수정 추가
```
String sql = "update Member set money=? where member_id = ?";
```
- MemberRepositoryV0Test - 회원 수정 추가
> money: 10000 > 20000 (memberV0)
- MemberRepositoryV0 - 회원 삭제 추가
```
String sql = "delete from Member where member_id = ?"
```
- MemberRepositoryV0Test - 회원 tkrwp 추가
```
assertThatThrownBy(()-> repository.findById(member.getMemberId()))
	.isInstanceOf(NoSuchElementException.class)
```
> assertThatThrownBy는 해당 예외가 발생해야 검증에 성공한다.
- cf
> 물론 테스트 중간에 오류가 발생해서 삭제 로직을 수행할 수 없다면 테스트를 반복해서 실행할 수 없다.
>> 트랜잭션을 활용하면 이 문제를 깔끔하게 해결할 수 있다.