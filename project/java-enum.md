## Java Enum활용 데이터 저장

---

프로젝트에서 외부 배치 모듈 설치해 웹소켓을 통해 DB를 체크해 insert가 되어있으면 해당 row를 체크해 데이터를 통신하는 방식을 썼습니다.  
자세하게는 외부에서 정해진 스펙에 따라 특정 row의 데이터 형식이 300Byte 내에서 지정된 값을 통해 데이터를 통신하는 구조였습니다.  
이 고정된 Length와 지정된 값을 DB에 보관해서 사용하자는 의견이 있었고 저는 이를 비효율 적이라 생각했습니다.  

**실제 운영 코드를 제외한 예제코드로 작성했습니다.**

### 문제점

---

 - 매번 DB를 통해서 고정된 값을 불러옴 
 - IDE, DB, 외부문서 세가지를 켜놓고 체크를 해야함 
 - 스펙 수정 발생시 Entity, 개발 서버 Table, 운영 서버 Table 수정

### 해결 방향

---

자료를 찾아 보던중 Enum을 활용한 글을 봤고 이를 사용해보면 좋겠다고 생각을 했고 아래와 같이 개선을 해볼 수 있다고 생각했습니다.  

1. DB가 아닌 객체로 관리
2. IDE 하나로 작업해 유지 보수성 증가
3. 스펙 수정 발생시 Enum만 수정

### 코드

---

#### GoodsRegister.java

```java
@Getter
public class GoodsRegister {

    private String name;
    private int price;
    private int stackQuantity;
    private String registerDate;

    public GoodsRegister(String name, int price, int stackQuantity, String registerDate) {
        this.name = name;
        this.price = price;
        this.stackQuantity = stackQuantity;
        this.registerDate = registerDate;
    }
}
```

#### GoodsRegisterEnum.java

```java
/**
 * C : 값 뒤로 빈값은 공백
 * N : 값 앞으로 빈값은 0,
 * D : 날짜 형식 입력 (ex :20031215) 아닐시 리턴 값 실패
 */
@Getter
public enum GoodsRegisterEnum implements DocumentEnum {
    GOODS_NAME("name", "C", 9, 0),
    GOODS_PRICE("price", "N", 10, 9),
    STACK_QUANTITY("stackQuantity", "C", 3, 19),
    REGISTER_DATE("registerDate", "D", 8, 22);

    private final String code;
    private final String type;
    private final int length;
    private final int start;

    GoodsRegisterEnum(String code, String type, int length, int start) {
        this.code = code;
        this.type = type;
        this.length = length;
        this.start = start;
    }
}
```

문서에 있던 개발에 필요한 정보 및 주의 사항을 간략하게 정리해서 Enum객체로 만들어 주고 Dto의 값과 연계하기 위해 Code 값을 지정했습니다.

---

#### DocumentEnum.java

```java
public interface DocumentEnum {
    String getCode();
    String getType();
    int getLength();
    int getStart();
}
```

#### DocumentUtils.java

```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public class DocumentUtils {

    private static final ObjectMapper mapper = new ObjectMapper();

    public static <T> HashMap<String, Object> toHashMap(T dtoClass) {
        return mapper.convertValue(dtoClass, new TypeReference<>() {}); // (1)
    }

    public static <T extends Enum<T> & DocumentEnum> String toFullText(Class<T> enumClass, HashMap<String, Object> map) {// (2)
        return EnumSet.allOf(enumClass).stream()
                .map(v -> createFullText(String.valueOf(map.get(v.getCode())), v.getLength(), v.getType()))
                .collect(Collectors.joining()); // (3)
    }

    private static String createFullText(String param, int len, String type) {
        String mask = type.equals("N") ? "0" : " ";
        String filter = fullTextFilter(param, len, mask);

        return type.equals("N") ? filter + param : param + filter;
    }

    private static String fullTextFilter(String param, int len, String mask) {
        return mask.repeat(Math.max(0, len - param.length())); // (4)
    }
}
```

(1) 순회시 시간 복잡도를 O(1)로 사용하기 위해 클라이언트 단 Dto를 HashMap으로 변경(타입 안정성을 위해서 TypeReference를 사용하였다.)
(2) 여러개의 문서가 생길 수 있으므로 DocumentEnum 인터페이스를 생성해서 사용했다.
(3) Enum을 순회하면서 클라이언트 값의 키값의 값이 있는지 체크하면서 전문을 작성
(4) 전문 작성시 타입별 마스킹 값을 셋팅

---

#### DocumentTest.java

```java
class DocumentTest {

    @Test
    void fullTextTest() {
        //given
        GoodsRegister goodsRegister = new GoodsRegister("500ml 물통", 10000, 999, "20180105");
        HashMap<String, Object> goodsRegisterMap = toHashMap(goodsRegister);

        //when
        String fulltext = toFullText(GoodsRegisterEnum.class, goodsRegisterMap);

        assertThat(fulltext).isEqualTo("500ml 물통 000001000099920180105");
    }
}
```
 테스트 코드 결과 총 30자의 문자열을 만들어낼 수 있다.

### 정리

---

DB 저장 값으로 사용시에는 Enum 객체보다 관리포인트가 많았고 DB툴, 문서, IDE 여러개를 켜두고 번갈아 보면서 작업 속도가 늘었을거 같다.   
Enum 객체 사용 후 위와 같이 한눈에 보기 편하게 만들 수 있었고 문서 변경시에도 Enum과 Dto 코드 값만 바꿔주면 바로 적용할 수 있기에 간편하게 사용할 수 있던거 같다.   
하지만 문서가 아닌 값이 자주 바뀌거나 개발자 외의 사람들이 수정 가능한 값이 되어버리면 Enum보다는 DB저장 방식으로 사용할 것 같다.
