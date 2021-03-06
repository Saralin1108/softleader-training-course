## MapStruct - Java bean mappings [WIP]

我們在專案中時常會需要複製 Java Bean, 也許是為了 Call Api 需要跟 DTO 互轉, 或是要轉成 VO 丟給某些頁面來使用, 簡單點我們會使用 `BeanUtils.copyProperties(target, source)` 來做, 但這類型的 `copyProperties` 幾乎都是透過 reflection 來存取 Object, 但用 reflection 的效能真的很慢, 這邊我們推薦使用 [MapStruct](https://mapstruct.org/) 來取代 `copyProperties`

MapStruct 基於 [JSR 269](https://www.jcp.org/en/jsr/detail?id=269) 來幫你產生對應的程式, 官方提到的優點包含了:

- 在 compile 的階段就可以知道程式撰寫錯誤, 不用等到 runtime
- 超級好的效能
- 沒有額外的 runtime lib 依賴需求

延伸閱讀: 

- [MapStruct Reference Guide](https://mapstruct.org/documentation/stable/reference/html/)
- [Performance of Java Mapping Frameworks](https://www.baeldung.com/java-performance-mapping-frameworks)
- [Java-JSR-269-插入式註解處理器](https://liuyehcf.github.io/2018/02/02/Java-JSR-269-%E6%8F%92%E5%85%A5%E5%BC%8F%E6%B3%A8%E8%A7%A3%E5%A4%84%E7%90%86%E5%99%A8/)


## Setup w/ Lombok

公司的專案大多都有使用 [Project Lombok](https://projectlombok.org/), MapStruct 亦可整合一起使用, 只要在 `pom.xml` 中加入依賴即可:

```xml
...
<properties>
    <mapstruct.version>1.3.1.Final</mapstruct.version>
    <lombok.version>1.18.0</lombok.version>
	<m2e.apt.activation>jdt_apt</m2e.apt.activation>
</properties>
...
<dependencies>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${mapstruct.version}</version>
    </dependency>
    <dependency>
      <groupId>org.mapstruct</groupId>
      <artifactId>mapstruct-processor</artifactId>
      <version>${mapstruct.version}</version>
      <optional>true</optional>
    </dependency>
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <version>${lombok.version}</version>
      <optional>true</optional>
    </dependency>
</dependencies>
...
```

> 請確保使用版本不可低於: MapStruct 1.2.0.Beta1 及 Lombok 1.16.14  
> 需注意, 有MapperClass存在的專案就必須親自引用 `mapstruct-processor`, 無論其專案下引用的jar是否曾經包含過, 否則不會正常compile出實作


## Softleader Guide

### 撰寫規範

要點:
1. Mapper 以 `@org.mapstruct.Mapper public interface Mapper` 宣告於所屬 class 下
2. Mapper Instance 使用 `Mapper INSTANCE = Mappers.getMapper(Mapper.class);` 不註冊至 spring
3. Method 的宣告, 以From的角度進行撰寫
4. Method Name 以 from, copy, update 為開頭進行宣告, 可依當下情境選擇

由於公司的Entity, Vo數量眾多, 為了方便管理, 以及避免重複造輪等理由, 建議將Mapper以InnerClass的形式進行撰寫在 Entity or Vo class內  
例:

```java
@Getter
@Setter
@ToString(callSuper = true)
public class FinancePayInfoRequest extends FinancePayInfoDto {

  /** 財務給付通知書 */
  private List<FinanceNoticeRequest> notices = Lists.newArrayList();

  /** 財務給付記錄 */
  private List<FinancePaidLogRequest> paidLogs = Lists.newArrayList();

  /** 財務受款人資料 */
  private List<FinanceReceiverInfoRequest> receiverInfos = Lists.newArrayList();

  /** 賠付對象資料 */
  private List<ClmPaymentRequest> payments = Lists.newArrayList();

  @org.mapstruct.Mapper
  public interface Mapper {
    Mapper INSTANCE = Mappers.getMapper(Mapper.class);

    @Mapping(target = "notices", ignore = true)
    @Mapping(target = "paidLogs", ignore = true)
    @Mapping(target = "receiverInfos", ignore = true)
    FinancePayInfoRequest from(FinancePayInfoData bean);
  }

  @org.mapstruct.Mapper
  public interface NoIdMapper {
    NoIdMapper INSTANCE = Mappers.getMapper(NoIdMapper.class);

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "createdTime", ignore = true)
    @Mapping(target = "modifiedBy", ignore = true)
    @Mapping(target = "modifiedTime", ignore = true)
    void update(FinancePayInfoData source, @MappingTarget FinancePayInfoRequest target);
  }

}
```
使用時
```java
FinancePayInfoRequest copyPropertiesToRequest(FinancePayInfoData data){
  return FinancePayInfoRequest.Mapper.INSTANCE.from(data);
}
```

### Samples

1. 型態變換 A to B
    ```java
    FooEntity from(FooVo source);
    ```

2. B 已存在的情況下, 將 A 的欄位複製 to B
    ```java
    void from(FooVo source, @MappingTarget FooEntity target);
    ```

3. 複製的過程中, 需要略過某些欄位 ignore `A.id`
    ```java
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "createdTime", ignore = true)
    @Mapping(target = "modifiedBy", ignore = true)
    @Mapping(target = "modifiedTime", ignore = true)
    FooEntity from(FooVo source);
   
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "createdTime", ignore = true)
    @Mapping(target = "modifiedBy", ignore = true)
    @Mapping(target = "modifiedTime", ignore = true)
    void update(FooVo source, @MappingTarget FooEntity target);
    ```

4. 要複製的欄位名稱不同 `A.localName` to `B.name`
    ```java
    @Mapping(source = "localName", target = "name")
    FooEntity from(FooVo source);
    ```

5. 需要略過的欄位在更下層的位置 ignore `A.B.id`
    ```java
    @Data
    class BooEntity {
      private Long id;
      private String name;
    }
   
    @Data
    class FooEntiy {
      private Long id;
      private List<BooEntity> boo;
   
      @org.mapstruct.Mapper
        public interface Mapper {
        Mapper INSTANCE = Mappers.getMapper(Mapper.class);
   
        FooEntiy from(FooVo source);
        List<BooEntity> from(List<BooVo> source);

        @Mapping(target = "id", ignore = true)
        BooEntity from(BooVo source);
      }
    }
    ```
    > 同一個Mapper下, 若欄位之間是有關連的, 會優先使用Mapper底下定義的複製方式
    > 但是前提條件是中間的每一層都必須寫出來, 且 List 跟 Bean 的複製要分開來各寫一次
    > FooEntity > List<BooEntity> > BooEntity

6. 由於DB雙向關聯的關係, 造成複製遞迴 `A.B.A.B.A.....`
    ```java
    
    @Data
    class BooEntity {
      private Long id;
      private FooEntity foo;
    }
   
    @Data
    class FooEntiy {
      private Long id;
      private List<BooEntity> boo;
   
      @org.mapstruct.Mapper
        public interface Mapper {
        Mapper INSTANCE = Mappers.getMapper(Mapper.class);

        FooEntiy from(FooVo source, @Context MapStructUtils.CycleAvoidingContext context);
      }
    }
    ```
    ```java
    public class MapStructUtils {
    
      /**
       * 避免遞迴的關係造成無限迴圈的Mapping
       * 物件內若有遞迴的關係發生時，可使用此class避免，使用細節參考下列網址範例
       * https://github.com/mapstruct/mapstruct-examples/tree/master/mapstruct-mapping-with-cycles
       */
      public static class CycleAvoidingContext {
        private Map<Object, Object> knownInstances = new IdentityHashMap<>();
    
        @BeforeMapping
        public <T> T getMappedInstance(Object source, @TargetType Class<T> targetType) {
          return (T) knownInstances.get( source );
        }
    
        @BeforeMapping
        public void storeMappedInstance(Object source, @MappingTarget Object target) {
          knownInstances.put( source, target );
        }
      }
    }
    ```
    > 這 class 在 jasmine-common 有
    
    使用:
    ```java
    FooEntity foo = FooEntity.Mapper.INSTANCE.from(fooVo, new MapStructUtils.CycleAvoidingContext());
    ```
    > 呼叫的時候, 將 `MapStructUtils.CycleAvoidingContext` new 出來
    > 需注意這個 instance 是不能共用的, 每當需要時就得 new 一個

7. 
