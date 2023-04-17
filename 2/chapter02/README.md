# [이론] 2. 데이터 접근 기술 - 스프링 JdbcTemplate
## 1. JdbcTemplate 소개와 설정
SQL을 직접 사용하는 경우에 스프링이 제공하는 JdbcTemplate은 아주 좋은 선택
- 장점
> 설정의 편리함
>> spring-jdbc 라이브러리에 포함, 복잡한 설정 없이 바로 사용
> 반복 문제 해결
>> 템플릿 콜백 패턴 사용하여 대부분의 작업을 대신 처리 (커넥션 획득, 결과 반복, 커넥션 종료, 트랜잭션, 예외 변환기)
- 단점
> !동적 SQL을 해결하기 어렵다

- JdbcTemplate 설정
> build.gradle: spring-boot-starter-jdbc, h2
- item table 생성

## 2. JdbcTemplate 적용 1 - 기본
- JdbcTemplateItemRepositoryV1
```
save() {
    KeyHolder keyHolder = new GeneratedKeyHolder();
    template.update(connnection -> {
        connection.prepareStatement(sql, new String[]{"id"});
        ~
    }, keyHolder);

    long key = keyHolder.getKey().longValue();
}
```
- 기본
> this.template = new JdbcTemplat(dataSource);  
- save()
> template.update()  
PK 생성에 identity(auto increment)방식, `KeyHolder`와`con.prepareStatement(sql, new String[]{"id"})`를 사용해서 id를 지정해주면 INSERT 쿼리 실행 이후에 데이터베이스에서 생성된 PK ID 값을 확인할 수 있다.
- update(): template.update()
- findById() 
> template.queryForObject, RowMapper
>> 결과가 업으면 EmptyResultDataAccessException, 둘 이상이면 IncorrectResultSizeDataAccessException
- findAll(): template.query()
- itemRowMapper(): ResultSet, JdbcTemplate이 루프를 돌려주고, 개발자는 RowMapper를 구현해서 내부 코드만 채운다고 이해하면 된다.

## 3. JdbcTemplate 적용 2 - 동적 쿼리 문제
- 상황에 따른 SQL을 동적으로 생성해야 된다.
> MyBatis의 가장 큰 장점은 SQL을 직접 사용할 때 동적 쿼리를 쉽게 작성할 수 있다는 점

## 4. JdbcTemplate 적용 3 - 구성과 실행
- JdbcTemplateV1Config, ItemServiceApplicaiton - 변경
- 데이터베이스 접근설정 - application.yml
- 로그 추가 - application.yml
> logging.level.orig.springframework.jdbc=debug

## 5. JdbcTemplate - 이름 지정 파라미터 1
- 순서대로 바인딩
> !SQL 코드 순서 변경시, 결과적으로 파라미터 바인딩 순서가 바뀌는 매우 심각한 문제 발생
- 이름 지정 바인딩
> `NamedParameterJdbcTemplate`
- JdbcTemplateItemRepositoryV2
```
save(Item item) {
    String sql = "insert into item(item_name, price, quantity) values (:itemName, :price, :quantity)";
    SqlParameterSource param = BeanPropertySqlParameterSource(item);
    template.update(sql, param, keyHolder);
}

update(ItemUpdateDto updateParam) {
    SqlParameterSource param = MapSqlParameterSource()
        .addValue("itemName", updateParam.getItemName())
        .addValue("price", updateParam.getPrice());
    template.update(sql, param);
}

findById(Long id) {
    Map<String, Object> param = Map.of("id", id);
    template.queryForObject(sql, param, itemRowMapper());
}

RowMapper itemRowMapper() {
    return BeanProperyRowMapper.newInstance(Item.class);
}
```
- 기본 this.template = new NamedParameterJdbcTemplate(dataSource);
- save()
> ? 대신에 :파라미터이름 을 받는 것을 확인할 수 있다.

## 6. JdbcTemplate - 이름 지정 파라미터 2
- 이름 지정 파라미터
> Map 처럼 key, value 데이터구조  
template.update(sql, param, keyHolder);  
- 종류 크게 3가지
> `Map`  
> `SqlParameterSource`
>> `MapSqlParameterSource`  
`BeanPropertySqlParameterSource`
- 1. Map
- 2. MapSqlParameterSource
> Map과 유사한데 SQL 타입 지정 등 SQL에 좀 더 특화된 기능을 제공  
메서드 체인 기능
- 3. BeanPropertySqlParameterSource
> 자바빈 프로퍼티 규약을 통해서 자동으로 파라미터 객체를 생성
- `BeanPropertyRowMapper`
> BeanPropertyRowMapper.newInstance(Item.class);  
ResultSet의 결과를 받아서 자바빈 규약에 맞추어 데이터를 변환한다. (실제로는 리플렉션 같은 기능을 사용한다.)
- 별칭
> 객체의 컬럼과 데이터베이스 컬럼이 다를 경우
- 관례의 불일치
> 카멜(JAVA), 스네이크(DB)  
BeanPropertyRowMapper는 언더스코어 표기법을 카멜로 자동 변환

## 7.JdbcTemplate - 이름 지정 파라미터 3
- JdbcTemplateV2Config, ItemServiceApplication - 변경

## 8.JdbcTemplate - SimpleJdbcTemplate
JdbcTemplate은 INSERT SQL 을 직접 작성하지 않아도 되도록 `SimpleJdbcInsert`라는 편리한 기능을 제공한다
- JdbcTemplateItemRepositoryV3
```
public JdbcTemplateItemRepositoryV2(DataSource dataSource) {
    this.jdbcInsert = new SimpleJdbcInsert(dataSource)
        .withTableName("item")
        .usingGeneratedKeyColumns("id)
        .usingColumns("item_name", "price", "quantity"); // 생략가능
}

save(Item item) {
    SqlParameterSource param = new BeanPropertySqlParameterSource(item);
    long key = jdbcInsert.executeAndReturnKey(param);
}
```
- 기본
> this.jdbcInsery = new SimpleJdbcInsert(dataSource);
- SimpleJdbcInsert
> withTableName: 데이터를 지정할 테이블 명  
usingGeneratedKeyColumns: key를 생성하는 PK 컬럼 명  
usingColumns: INSERT SQL 에 사용할 컬럼을 지정, 생략 가능
>> SimpleJdbcInsert는 생성 시점에 데이터베이스 테이블의 메타 데이터를 조회한다, 따라서 어떤 컬럼이 있는지 확인 할 수 있으므로 usingColumns을 생략할 수 있다.
- JdbcTemplateV3Config, ItemServiceApplication - 변경

## 9.JdbcTemplate 기능 정리
- 주요 기능
> JdbcTemplate  
NamedParameterJdbcTemplate  
SimpleJdbcInsert  
SimpleJdbcCall
>> SimpleJdbcCall 메뉴얼  
https://docs.spring.io/spring-framework/docs/current/reference/html/dataaccess.html#jdbc-simple-jdbc-call-1

- JdbcTemplate 사용법 정리
https://docs.spring.io/spring-framework/docs/current/reference/html/dataaccess.html#jdbc-JdbcTemplate
> 조회
>> 단건 조회 - 숫자 조회  / 문자 조회 / 객체 조회
>>> jdbcTemplate.queryForObject("select count(*) from dual, Integer.class)  
>> 목록 조회 - 객체
>>> jdbcTemplate.query(sql, rowMapper());
> 변경(INSERT, UPDATE, DELETE)
>> jdbcTemplate.update()
> 기타 기능
>> 임의의 SQL 을 실행할 때는 execute() 사용, DDL, 스토어드 프로시저 등

## 10. 정리
실무에서 가장 간단하고 실용적인 방법  
JPA와 같은 ORM 기술을 사용하면서 동시에 SQL을 직접 작성해야 할 때가 있는데 그때도, JdbcTemplate을 함께 사용하면 된다.
> !JdbcTemplate 최대 단점, 동적 쿼리 문제를 해결하지 못한다는 점
>> 동적 쿼리 문제를 해결하면서 동시에 SQL도 편리하게 작성할 수 있게 도와주는 기술이 MyBatis