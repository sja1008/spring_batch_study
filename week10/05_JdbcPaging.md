# JdbcPagingItemReader, JdbcBatchITemWriter
## JdbcPagingItemReader 개요
- 페이징 처리를 지원하는 ItemReader
## JdbcPagingItemReader 주요 구성 요소
- DataSource
  - 데이터베이스 연결 정보 설정
- SqlQuery
  - 데이터를 읽을 SQL 쿼리 설정
- RowMapper
  - SQL 쿼리 결과를 Item으로 변환하는 역할
- PageSize
  - 페이지 크기 설정
- SkippableItemReader
  - 오류 발생시 해당 item을 스킵할지 여부 설정
- ReadListener
  - 읽기 시작, 종료, 오류 발생 등의 이벤트 처리
- SaveStateCallback
  - 잡 중단시 현재 상태를 저장
  - 재시작 시 이어서 처리
## JdbcPagingItemReader 샘플 코드
- Customer 클래스 생성
``` 
public class Customer {
    private String name;
    private int age;
    private String gender;
} 
```
- 쿼리 Provider 생성하기
  - 실제 batch를 위해서 데이터를 읽어올 쿼리를 작성
```
@Bean
public PagingQueryProvider queryProvider() {
    SqlPagingQueryProviderFactoryBean queryProvider = new SqlPagingQueryProviderFactoryBean();
    queryProvider.setDataSource(dataSource);
    queryProvider.setSelectClause("id, name, age, gender");
    queryProvider.setFromClause("from customer");
    queryProvider.setWhereClause("where age >= :age");
    
    Map<String, Order> sortKeys = new HashMap<>(1);
    sortKeys.put("id", Order.DESCENDING);
    
    queryProvider.setSortKeys(sortKeys);
    
    return queryProvider.getObject();    
}
```
- JdbcPagingItemReader 작성하기
``` 
@Bean
public JdbcPagingItemReader<Customer> pagingItemReader() {
    Map<String, Object> parameterValues = new HashMap<>();
    parameterValues.put("age", 20);
    
    return new JdbcPagingItemReaderBuilder<Customer>()
        .name("jdbcPagingItemReader")
        .fetchSize(CHUNK_SIZE)
        .dataSource(dataSource)
        .rowMapper(new BeanPropertyRowMapper<>(Customer.class))
        .queryProvider(queryProvider())
        .parameterValues(parameterValues)
        .build();
} 
```
## JdbcBatchItemWriter 개요
- JDBC를 통해 데이터베이스에 저장하는 ItemWriter
## JdbcBatchItemWriter 주요 구성 요소
- DataSource
  - 데이터베이스 연결 정보 설정
- SqlStatementCreator
  - INSERT 쿼리를 생성하는 역할
- PreparedStatementSetter
  - INSERT 쿼리의 파라미터를 설정하는 역할
- ItemSqlParameterSourceProvider
  - Item객체를 기반으로 PreparedStatement에 전달할 파라미터 값을 생성하는 역할
### 장점
- JDBC를 통해 다양한 데이터베이스 데이터를 저장할 수 있음
- 성능 : 대량의 데이터를 빠르게 저장할 수 있음
- 유연성 : 다양한 설정을 통해 원하는 방식으로 데이터를 저장할 수 있음
### 단점
- 설정 복잡성 : JDBC 설정 및 쿼리 작성이 복잡할 수 있음
- 데이터베이스 종속 : 특정 데이터베이스에 종속적
- 오류 가능성 : 설정 오류 시 데이터 손상 가능성
## JdbcBatchItemWriter 샘플 코드
- application.yaml 파일 작성하기
``` 
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/testdb?useUnicode=true&characterEncoding=utf8
    username: 
    password: 
  batch:
    job:
      name: JDBC_BATCH_WRITER_CHUNK_JOB
```
- 테이블 생성하기
``` 
create table testdb.customer2 
(
  id int auto_increment primary key,
  name varchar(100) null,
  age int null,
  gender varchar(10) null
); 
```
- JdbcBatchItemWriter 작성하기
```
@Bean
public JdbcBatchItemWriter<Customer> jdbcBatchItemWriter() {
    return new JdbcBatchItemWriterBuilder<Customer>()
        .dataSource(dataSource)
        .sql("INSERT INTO customer2 (name, age, gender) VALUES (?, ?, ?)")
        .itemSqlParameterSourceProvider(new BeanPropertyItemSqlParameterSourceProvider<>())
        .build();
```
- SqlParameterSourceProvider 작성하기
``` 
public class CustomerItemSqlParameterSourceProvider implements ItemSqlParameterSourceProvider<Customer> {
    @Override
    public SqlParameterSource createSqlParameterSource(Customer item) {
      return new BeanPropertySqlParameterSource(item);
    }
```
