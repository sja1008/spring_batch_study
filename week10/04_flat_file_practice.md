# FlatFileItemReader, FlatFileItemWriter 실습
## FlatFileItemReader 개요
- ItemReader 인터페이스를 구현하는 클래스
- 텍스트 파일로부터 데이터를 읽음
- 설정 및 사용이 간편
- 대규모 데이터 처리에도 효율적
- 다양한 텍스 파일 형식 지원
  - 고정 길이, 구분자 기반, 멀티라인 등
- 확장 가능성
  - 토크나이저, 필터 등
- 사용처
  - 고정 길이, 구분자 기반, 멀티라인 등
- 장점 : 강단하고 효율적, 다양한 텍스트 파일 형식 지원
- 단점 : 복잡한 데이터 구조 처리에 부적합
## FlatFileItemReader 주요 구성 요소
- Resource: 읽을 텍스트 파일
- LineMapper: 텍스트 파일의 각 라인을 Item으로 분리하는 역할
- LineTokenizer: 텍스트 파일의 각 라인을 토큰으로 분리하는 역할
- FieldSetMapper: 토큰을 Item의 속성에 매핑하는 역할
- SkippableLineMapper: 오류 발생시 해당 라인 스킵
- LineCallbackHandler: 라인별로 처리 수행
- ReadListener: 읽기 시작, 종료, 오류 발생 등의 이벤트 처리
## FlatFileItemWriter 개요
- ItemWriter 인터페이스를 구현하는 클래스
- 데이터를 텍스트 파일로 출력
## FlatFileItemWriter 주요 구성 요소
- Resource: 츨력 파일 경로를 지정
- LineAggregator: Item을 문자열로 변환하는 역할
- HeaderCallback: 출력 파일 헤더를 작성하는 역할
- FooterCallback: 출력 파일 푸터를 작성하는 역할
- Delimiter: 항목 사이 구분자 지정
- AppendMode: 기존 파일에 추가할지 여부 지정
- 장점
  - 쉽고 유연하게 출력 파일을 만들 수 있음
  - 대량의 데이터를 빠르게 출력할 수 있음
- 단점
  - 텍스트 파일 형식만 지원
  - 복잡한 구조의 데이터를 출력할 경우 설정이 복잡해짐
  - 설정 오류시 출력 파일 손상
## 샘플 코드
### configuration 코드
``` 
@Configuration
public class FlatFileItemJobConfig {
    public static final int CHUNK_SIZE = 100;
    public static final String ENCODING = "UTF-8";
    public static final String FLAT_FILE_WRITER_CHUNK_JOB = "FLAT_FILE_WRITER_CHUNK_JOB";

    private ConcurrentHashMap<String, Integer> aggregateInfos = new ConcurrentHashMap<>();
    private final ItemProcessor<Customer, Customer> itemProcessor = new AggregateCustomerProcessor(aggregateInfos);

    @Bean
    public Job flatFileJob(Step flatFileStep, JobRepository jobRepository) {
        log.info("------------------ Init flatFileJob -----------------");
        return new JobBuilder(FLAT_FILE_WRITER_CHUNK_JOB, jobRepository)
                .incrementer(new RunIdIncrementer())
                .start(flatFileStep)
                .build();
    }

    @Bean
    public Step flatFileStep(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
        log.info("------------------ Init flatFileStep -----------------");
        return new StepBuilder("flatFileStep", jobRepository)
                .<Customer, Customer>chunk(CHUNK_SIZE, transactionManager)
                .reader(flatFileItemReader())
                .processor(itemProcessor)
                .writer(flatFileItemWriter())
                .build();
    }
    
    @Bean
    public FlatFileItemReader<Customer> flatFileItemReader() {
        return new FlatFileItemReaderBuilder<Customer>()
                .name("FlatFileItemReader")
                .resource(new ClassPathResource("./customer.csv"))
                .encoding(ENCODING)
                .delimited().delimiter(",")
                .names("name", "age", "gender")
                .targetType(Customer.class)
                .build();
    }

    @Bean
    public FlatFileItemWriter<Customer> flatFileItemWriter() {
        return new FlatFileItemWriterBuilder<Customer>()
                .name("flatFileItemWriter")
                .resource(new FileSystemResource("./output/customer_new.csv"))
                .encoding(ENCODING)
                .delimited().delimiter("\t")
                .names("Name", "Age", "Gender")
                .append(false)
                .lineAggregator(new CustomerLineAggregator())
                .headerCallback(new CustomerHeader())
                .footerCallback(new CustomerFooter(aggregateInfos))
                .build();
    }
}
```
### FlatFileItemReader 샘플 코드
- Customer 모델 생성
``` 
public class Customer {
    private String name;
    private int age;
    private String gender;
}
```
- FileFlatITemReader 빈 생성
``` 
@Bean
public FlatFileItemReader<Customer> flatFileItemReader() {
    return new FlatFileItemReaderBuilder<Customer>()
        .name('FlatFileItemReader")
        .resouce(new ClassPathResource("./customer.csv"))
        .encoding(ENCODING)
        .delimited().delimiter(",")
        .names("name", "age", "gender)
        .targetType(Customer.class)
        .build()
}
```
### FlatFileItemWriter 샘플 코드
- FlatFileItemWriter 작성하기
``` 
@Bean
public FlatFileItemWriter<Customer> flatFileItemWriter() {
    return new FlatFileItemWriterBuilder<Customer>()
        .name("flatFileItemWriter")
        .resource(new FileSystemResource("./customer_new.csv"))
        .encoding(ENCODING)
        .delimited().delimiter("\t")
        .names("Name", "Age", "Gender")
        .append(false)
        .lineAggregator(new CustomerLineAggregator())
        .headerCallback(new CustomerHeader())
        .footerCallback(new CustomerFooter(aggregateInfos))
        .build();
}
```
- CustomerLineAggregator 작성하기
``` 
public class CustomerLineAggregator implements LingAggregator<Customer>{
    @Override
    public String aggregate(Customer item) {
        return item.getName() + "," + item.getAge();
    }
}
```
- CustomerHeader 작성하기
``` 
public class CustomerHeader implements FlatFileHEaderCallback {
    @Override
    public void writeHeader(Writer writer) {
        writer.write("ID,AGE");
    }
}
```
- CustomerFooter 작성하기
``` 
public class CutomerFooter implements FlatFileFooterCallback {
    ConcurrentHashMap<String, Integer> aggregateCustomers;
    
    public CustomerFooter(ConcurrentHashMap<String, Integer> aggregateCustomers) {
        this.aggregateCustomers = aggregateCustomers;
    }
    
    @Override
    public void writeFooter(Writer writer) throws IOException {
        writer.write("총 고객 수 : " + aggregateCustomers.get("TOTAL_CUSTOMERS"));
        writer.write(System.lineSeperator())
        writer.write("총 나이: " + aggregateCustomers.get("TOTAL_AGES"));
    }
}
```
- AggregateCustomerProcessor 작성하기
``` 
public class AggregateCustomerProcessor implements ItemProcessor<Customer, Customer> {
    ConcurrentHashMap<String, Integer> aggregateCustomers;
    
    public AggregateCustomerProcessor(ConcurrentHashMap<String, Integer> aggregateCustomers) {
        this.aggregateCunstomers = aggregateCustomers;
    }
    
    @Override
    public Customer process(Customer item) throws Exception {
        aggregateCunstomers.putIfAbsent("TOTAL_CUSTOMERS", 0);
        aggregateCunstomers.putIfAbsent("TOTAL_AGES", 0);
        
        aggregateCunstomers.put("TOTAL_CUSTOMERS", aggregateCustomers.get("TOTAL_CUSTOMERS") + 1);
        aggregateCunstomers.put("TOTAL_AGES", aggregateCustomers.get("TOTAL_AGES") + item.getAge());
        
        return item;
    }
}
```






