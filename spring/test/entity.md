## Entity 테스트

Entity는 equals로 데이터를 비교할 때 id로만 검증을 진행해야 한다.
테스트에서 repository를 모킹할 때, save 함수같은 경우 entity를 그대로 전해주므로 id가 null이어서 mocking이 항상 실패한다.

유효한 방법은
1. @SpringBootTest 어노테이션을 테스트에 붙여서 in-memory db로 데이터베이스도 활성화해서 repository를 mocking하지 않고 테스트한다.<br/> 
다만 이렇게 진행할 경우 실제 Spring이 실행되므로 테스트가 많이 느려진다
2. 실제 Entity를 건네지 않고, Repository에서 연관된 데이터를 전체 받아서 Entity를 내부에서 만들어서 save를 호출한다<br/>
ORM를 사용할 때는 그다지 권장하지 않는 방법이다
3. mocking시 any()를 사용해서 파라미터 검증을 건너뛰고, 결과값으로 인자를 그대로 돌려준다.<br/>
save를 호출했을 때 건네준 데이터가 그대로 반환될거고, Service는 해당 데이터를 돌려준다.<br/>
이 때 출력 테스트를 진행하려면 Service에서 Entity를 반환하지 않고, 자체적으로 DTO를 만들어서 돌려줘야 equals 비교가 가능하다.<br/>
최종적으로 verify로 save 함수가 호출되었는지 비교하고, 출력값도 같이 검증하면 된다.

```kotlin
// kotest와 mockk를 사용했을 때 예제
feature("이름고 설명을 생성한다") {
    scenario("사용자가 정보를 입력하면 해당 정보를 기반으로 데이터를 생성한다") {
      // given
      val params = NameDescription(
        name = "name",
        description = "desc",
      )

      every { repository.save(any()) } returnsArgument 0

      // when
      val result = sut.create(params)

      // then
      verify(exactly = 1) { repository.save(any()) }
      result shouldBe NameDescriptionResult(name = "name", description = "desc")
    }
  }
```

Entity의 id로 비교하는 속성때문에 service 레이어에서는 entity를 그대로 반환해주면 안되게 강제된다.<br/>
또, repository에서 entity를 파라미터로 그대로 받을 경우에, mocking시 any()일경우 이외에는 검증이 불가능해진다.<br/>

Entity를 비교할 때 id가 null이면 사용할 속성을 Annotation으로 특정해서 (@Comparable 같은 커스텀) reflection으로 전체 가져와서 비교 검증을 진행하던가 아니면 위에 방식으로 진행해야 한다.

Annotation으로 특정하는 방법도 매 번 해당 annotation을 적어줘야하고, 실수로 비교할 필드에 추가하지 않으면 equals비교가 실패해야하는데 성공하게 되므로 믿음의 영역이 된다.<br/>
JPA를 쓸 경우 시간을 소요하는 대신 적절한 테스트 검증을 위해서 Repository를 사용하는 Service는 SpringBootTest를 사용하던가<br/>
위의 방식대로 테스트를 진행해야 할 것 같다.
