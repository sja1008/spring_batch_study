# MyBatisPagingItemReader / MyBatisBatchItemWriter
## MyBatisPagingItemReader
- ItemReader 인터페이스를 구현하는 클래스
- 장점
  - 간편한 설정
  - 쿼리 최적화
  - 동적 쿼리 지원
- 단점
  - MyBatis 의존성
  - Chunk-oriented Processing 방식과 비교했을때 커스터마이징이 더 복잡
## 주요 구성 요소
- SqlSessionFactory
  - SqlSessionFactory를 사용하여 데이터를 읽음
- QueryId
  - MyBatis 쿼리 ID
- PageSize
  - pageSize를 이용하여 offset, limit을 이용하는 기준을 설정
- SkippableItemReader
  - 오류 발생시 해당 item을 스킵할지 여부 설정
- ReadListener
  - 읽기 시작, 종료, 오류 발생 등의 이벤트를 처리
- SaveStateCallback
  - 잡 중단시 현재 상태를 저장
## MyBatisBatchItemWriter
- ItemWriter 인터페이스를 구현하는 클래스
- 장점
  - 다양한 데이터베이스 연동
  - SQL 쿼리 분리
  - 유연성
- 단점
  - 설정 복잡
  - 데이터베이스 종속
  - 오류 가능성
## 주요 구성 요소
- SqlSessionTemplate
  - SqlSession 생성 및 관리를 위한 템플릿 객체
- SqlSessionFactory
  - SqlSessionTemplate을 생성하기 위한 팩토리 객체
- StatementId
  - MyBatis 쿼리 ID
- ItemToParameterConverter
  - 객체를 ParameterMap으로 변경할 수 있음
## 구현 코드
- DB에서 데이터를 읽어서 임베딩 시킨 후 다시 DB에 저장하는 예제
### TableMetadata 클래스
``` 
@Getter
@Setter
public class TableMetadata {
    String tableName;
    String description;
    float[] embedding;
}
```
### MyBatisPagingItemReader 작성
```
@Bean
@StepScope
public MyBatisPagingItemReader<TableMetadata> tableMetadataReader(
        @Value("${datasource.source.url}") String url,
        @Value("${datasource.source.username}") String username,
        @Value("${datasource.source.password}") String password,
        @Value("${datasource.source.driver-class-name}") String driverClassName

) throws Exception {
   
    return new MyBatisPagingItemReaderBuilder<TableMetadata>()
            .sqlSessionFactory(
                    createSqlSessionFactory(
                            createDataSource(url, username, password, driverClassName)
                    )
            )
            .pageSize(PAGE_SIZE)
            .queryId("fetchTableMetadata")
            .build();
}
```
### read query 작성하기
- _skiprows와 _pagesize는 MyBatisPagingItemReader에서 자동으로 넣어줌
- _skiprows: 페이지 시작 위치
- _pagesize: 페이지 크기
``` 
<mapper namespace="kr.study.batch.mapper.TableMetadataMapper">
    <select id="fetchTableMetadata" resultType="kr.study.batch.vo.TableMetadata">
        SELECT
            tbl_nm AS tableName,
            tbl_korn_nm AS description
        FROM t_mf_mng_schm_tbl
        where meta_mng_inst_id = 6
            	and info_sys_id = 2
            	and db_id = 2
        OFFSET #{_skiprows}
        LIMIT #{_pagesize}
    </select>
</mapper>
```
### Processor bean 만들기
- description을 임베딩 시킨 후 다시 저장
``` 
@Bean
@StepScope
public ItemProcessor<TableMetadata, TableMetadata> processor(EmbeddingService embeddingService) {
    log.info("===== processor =====");
    final AtomicInteger counter = new AtomicInteger(0);
    return item -> {
        item.setEmbedding(embeddingService.embedding(item.getDescription()));
        log.info("{}] processor, item : {}", counter.getAndIncrement(), item);
        return item;
    };
}
```
### MyBatisBatchItemWriter 작성
```
@Bean
@StepScope
public MyBatisBatchItemWriter<TableMetadata> tableMetadataWriter(
        @Value("${datasource.target.url}") String url,
        @Value("${datasource.target.username}") String username,
        @Value("${datasource.target.password}") String password,
        @Value("${datasource.target.driver-class-name}") String driverClassName
) throws Exception {
    log.info("===== writer =====");
    return new MyBatisBatchItemWriterBuilder<TableMetadata>()
            .sqlSessionFactory(
                    createSqlSessionFactory(
                            createDataSource(url, username, password, driverClassName)
                    )
            )
            .statementId("insertTableMetadata")
            .build();
}
```
### write query 작성하기
```
<mapper namespace="kr.study.batch.mapper.TableMetadataMapper">
    <insert id="insertTableMetadata">
        INSERT INTO embedding.schema_meta
        (tbl_nm, description, embedding)
        VALUES( #{tableName}, #{description}, #{embedding} );
    </insert>
</mapper>
```