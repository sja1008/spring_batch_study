# JpaPagingItemReader / JpaItemWriter
## JpaPagingItemReader 개요
- SpringBatch에서 제공하는 ItemReader
- JPA를 이용하여 데이터를 페이지 단위로 읽어올때 사용
- 쿼리 최적화: JPA 쿼리 기능을 사용하여 최적화된 데이터 읽기가 가능
- 커서 제어: JPA Creteria API를 사용하여 순회 제어 가능
## JpaPagingItemReader 구성 요소
- EntityManagerFactory
  - JPA 엔터티 매니저 팩토리 설정
- JpaQueryProvider
  - 데이터를 읽을 JPA 쿼리 제공
- PageSize
  - 페이지 크기 설정
- SkipableItemReader
  - 오류 발생시 해당 item을 스킵할지 여부 설정
- ReadListener
  - 읽기 시작, 종료, 오류 발생 등의 이벤트 처리
- SaveStateCallback
  - 잡 중단시 현재 상태를 저장
  - 재시작 시 이어서 처리
## 구현
### 엔터티 클래스 정의
``` 
@Getter
@Entity
@Table(name = "batch_job_execution")
public class BatchJobExecution {
    @Id
    private Long jobExecutionId;
    private Long version;
    private Long jobInstanceId;
    private LocalDateTime createTime;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    @Enumerated(EnumType.STRING)
    private BatchStatus status;
    private String exitCode;
    private String exitMessage;
    private LocalDateTime lastUpdated;
}
```
### JpaPagingItemReader 정의
- 방법 1 - builder 사용
```
@Bean
public JpaPagingItemReader<BatchJobExecution> batchJobExecutionJpaPagingItemReaderBuilder() throws Exception {
    return new JpaPagingItemReaderBuilder<BatchJobExecution>()
        .queryString("SELECT je FROM BatchJobExecution je WHERE je.exitCode = :exitCode order by je.lastUpdated desc")
        .entityManagerFactory(entityManagerFactory)
        .pageSize(CHUNK_SIZE)
        .parameterValues(Collections.singletonMap("exitCode", ExitStatus.FAILED))
        .build();
}
```
- 방법 2 - 생성자 사용
``` 
@Bean
public JpaPagingItemReader<BatchJobExecution> batchJobExecutionJpaPagingItemReader() throws Exception {
    JpaPagingItemReader<BatchJobExecution> jpaPagingItemReader = new JpaPagingItemReader<>();
    jpaPagingItemReader.setQueryString(
            "SELECT je FROM BatchJobExecution je WHERE je.exitCode = :exitCode order by je.lastUpdated desc"
    );
    jpaPagingItemReader.setEntityManagerFactory(entityManagerFactory);
    jpaPagingItemReader.setPageSize(CHUNK_SIZE);
    jpaPagingItemReader.setParameterValues(Collections.singletonMap("exitCode", ExitStatus.FAILED.getExitCode()));
    return jpaPagingItemReader;
} 
```

## JpaItemWriter 개요
- SpringBatch에서 제공하는 ItemWriter
- JPA를 이용하여 데이터를 저장하는데 사용
- 장점
  - ORM 연동
  - 객체 매핑
  - 유연성
- 단점
  - 설정 복잡성
  - 오류 가능성
## JpaItemWriter 구성 요소
- EntityManagerFactory
  - JPA 엔터티 매니저 팩토리 설정
- JpaQueryProvider
  - 데이터를 저장할 JPA 쿼리 제공
## 구현
### 엔터티 클래스 정의
```
@Getter
@Setter
@Entity
@Table(name = "batch_job_execution_backup")
public class BatchJobExecutionBackup {
    @Id
    private String backupId;
    private Long jobExecutionId;
    private Long version;
    private Long jobInstanceId;
    private LocalDateTime createTime;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    @Enumerated(EnumType.STRING)
    private BatchStatus status;
    private String exitCode;
    private String exitMessage;
    private LocalDateTime lastUpdated;
}
```
### JpaItemWriter 정의
``` 
@Bean
public JpaItemWriter<BatchJobExecutionBackup> jpaItemWriter() {
    return new JpaItemWriterBuilder<BatchJobExecutionBackup>()
        .entityManagerFactory(entityManagerFactory)
        .usePersist(true)
        .build();
}
```
## 실습
- [](https://github.com/sajacaros/spring-batch-jpa)