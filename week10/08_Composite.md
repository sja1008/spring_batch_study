# CompositeItemProcessor
## 개요
- Spring Batch에서 제공하는 ItemProcessor 인터페이스를 구현하는 클래스
- 여러 개의 ItemProcessor를 하나의 Processor로 연결하여 여러 단계의 처리 수행
## 주요 구성 요소
- Delegates: 처리를 수행할 ItemProcessor 목록
- TransactionAttribute: 트랜잭션 속성 설정
## 장점
- 단계별 처리
- 재사용 가능성
  - Processor를 재사용하여 다른 Job에서도 활용 가능
- 유연성
  - 다양한 ItemProcessor를 조합하여 원하는 처리 과정 구현 
## 단점
- 설정 복잡성
- 성능 저하
## 샘플 코드
### LowerCaseItemProcessor 작성하기
``` 
public class LowerCaseItemProcessor implements ItemProcessor<Customer, Customer> {
    @Override
    public Customer process(Customer item) throws Exception {
        item.setName(item.getName().toLowerCase());
        item.setGender(item.getGender().toLowerCase());
        return item;
    }
}
```
### After20YearsItemProcessor 작성하기
``` 
public class After20YearsItemProcessor implements ItemProcessor<Customer, Customer> {
    @Override
    public Customer process(Customer item) throws Exception {
        item.setAge(item.getAge() + 20);
        return item;
    }
}
```
### CompositeItemProcess 구현하기
``` 
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