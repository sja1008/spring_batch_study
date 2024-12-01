## SpringBatch 08: CompositeItemProcessor 으로 여러단계에 걸쳐 데이터 Transform하기

### CompositeItemProcessor: ItemProcessor 인터페이스 구현체
> 여러 개의 ItemProcessor를 하나의 Processor로 연결해 여러 단계의 처리 수행

- Delegates: 처리를 수행할 ItemProcessor 목록
- TransactionAttribute: 트랜잭션 속성 설정

장점
1. 단계별 처리: 여러 단계로 나누어 처리를 수행하므로 코드를 명확하고 이해하기 쉽게 만들어줌
2. 재사용: 각 단계별 Processor를 재사용하여 다른 Job에서도 가져다 사용 가능
3. 유연성: 다양한 ItemProcessor를 조합해 원하는 처리 과정 구현

단점
1. 설정 복잡성: 여러 개의 Processor를 설정하고 관리해야 하므로 설정이 복잡해질 수 있음
2. 성능 저하: 여러 단계의 처리 과정을 거치므로 성능 저하 가능성

### 실습 주요 내용
``` java
/* 이름과 성별을 소문자로 변경하는 item processor */
public class LowCaseItemProcessor implements ItemProcessor<Customer, Customer> {
    @Override
    public Customer process(Customer item) {
        item.setName(item.getName().toLowerCase());
        item.setGender(item.getGender().toLowerCase());
        return item;
     }
}
/* 현재 나이에 20을 더하는 item processor */
public class After20YearsItemProcessor implements ItemProcessor<Customer, Customer> {
    @Override
    public Customer process(Customer item) {
        item.setAge(item.getAge() + 20);
        return item;
    }
}
```
#### CompositeItemProcessor
빌더를 이용하여 delegates()로 위에 작성한 item processor를 전달
``` java
 @Bean
    public CompositeItemProcessor<Customer, Customer> compositeItemProcessor() {
        return new CompositeItemProcessorBuilder<Customer, Customer>()
                .delegates(List.of(
                        new LowerCaseItemProcessor(),
                        new After20YearsItemProcessor()
                ))
                .build();
}
```
#### output.csv
이름과 성별이 소문자로 변경되고 나이는 + 20살이 된 결과가 저장됨

CompositeItemProcessor의 리스트에 저장된 item processor 순으로 적용하여 데이터를 변경할 할 수 있음
