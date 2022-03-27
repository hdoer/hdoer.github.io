---
title: "JsonValue 使用说明"
# subtitle: "Java Jackson JsonValue 注解使用说明"
layout: post
author: "Wentao Dong"
date: 2021-07-11 21:00:00
catalog: true
header-style: post
header-img: "img/city_night.png"
tags:
  - Java
  - Jackson
  - Annotation
  - JsonValue
---

#### Jackson 版本

- 2.6.7

#### JsonValue说明

- @JsonValue 用于指定对象序列化时使用哪个方法的返回值作为序列化结果
- @JsonValue 标注在某个方法上，该方法不能有参数、返回值可以是任何的可序列化类型比如：Collection、Map、Bean但不能是void
- 同一个类中，@JsonValue注解只能标注在一个方法上
- 通常和@JsonCreator 注解一起使用，@JsonValue负责控制序列化，@JsonCreator负责控制反序列化
- @JsonValue 用在枚举类的方法上时，将用此值进行序列化和反序列化，原理是枚举类型是常量具有确定的映射关系。jackson构造了一个map， 使用方法返回值作为key，枚举作为value。
- @JsonValue 只有一个属性，标明是否激活该注解，主要结合覆盖（mix-in）注解使用。通常不需要设置该值，省略即可

#### Use Case

```java
// @JsonValue @JsonCreator 配合使用1
public class TextAnswer {
    @Getter
    private String id;
    @Getter
    private String answer;

    public TextAnswer() {}

    @JsonCreator
    public TextAnswer(Map<String, String> jsonMap) {
        this.id = jsonMap.get("id");
        this.answer = jsonMap.get("answer");
    }

    @JsonValue // 属性值默认为true
    public Map<String, String> toJsonMap() {
        HashMap<String, String> hashMap = new HashMap<>();
        hashMap.put("id", "test");
        hashMap.put("answer", "hello world!");
        return hashMap;
    }
}
public class TestMain {
    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    public static void main(String[] args) throws IOException {
        Test test = new Test(new TextAnswer());

        String testJsonStr = toJsonString(test);
        System.out.println(testJsonStr);

        test = OBJECT_MAPPER.readValue(testJsonStr, Test.class);
        System.out.println(test.getAnswer().getAnswer());
    }

    private static String toJsonString(Object o) throws JsonProcessingException {
        return OBJECT_MAPPER.writeValueAsString(o);
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    private static class Test{
        private TextAnswer answer;
    }
}
// output
// {"answer":{"answer":"hello world!","id":"test"}}
// hello world!
```

```java
// @JsonValue @JsonCreator 配合使用2
@Getter
@Setter
public class TextAnswer {
    private String id;
    private String answer;

    public TextAnswer() {}

    @JsonCreator
    public TextAnswer(Map<String, String> jsonMap) {
        this.id = jsonMap.get("id");
        this.answer = jsonMap.get("answer");
    }

    @JsonValue(value = false) // 属性值设为false
    public Map<String, String> toJsonMap() {
        HashMap<String, String> hashMap = new HashMap<>();
        hashMap.put("id", "test");
        hashMap.put("answer", "hello, world!");
        return hashMap;
    }
}

public class TestMain {
    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    public static void main(String[] args) throws IOException {
        TextAnswer textAnswer = new TextAnswer();
        textAnswer.setId("test.main.id");
        textAnswer.setAnswer("test.main: hello, world!");
        Test test = new Test(textAnswer);

        String testJsonStr = toJsonString(test);
        System.out.println(testJsonStr);

        test = OBJECT_MAPPER.readValue(testJsonStr, Test.class);
        System.out.println(test.getAnswer().getAnswer());
    }

    private static String toJsonString(Object o) throws JsonProcessingException {
        return OBJECT_MAPPER.writeValueAsString(o);
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    private static class Test{
        private TextAnswer answer;
    }
}

// output:
// {"answer":{"id":"test.main.id","answer":"test.main: hello, world!"}}
// test.main: hello, world!
```

```java
// @JsonValue 用于枚举类型
public class TestMain {
    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    public static void main(String[] args) throws IOException {
        TestType testType = new TestType(AnswerType.TEXT);

        String testJsonStr = toJsonString(testType);
        System.out.println(testJsonStr);

        testType = OBJECT_MAPPER.readValue(testJsonStr, TestType.class);
        System.out.println(testType.getAnswerType());
    }

    private static String toJsonString(Object o) throws JsonProcessingException {
        return OBJECT_MAPPER.writeValueAsString(o);
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    private static class Test{
        private TextAnswer answer;
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    private static class TestType{
        private AnswerType answerType;
    }

    @AllArgsConstructor
    private enum AnswerType {
        TEXT("text", "文本类型答案"), PIC("picture", "图片类型答案");
        private String name;
        private String desc;

        @JsonValue
        public String getType(){
            return name;
        }
    }
}
```

```java
// @JsonValue with Mix-in
@Getter
@Setter
public class TextAnswer {
    private String id;
    private String answer;

    public TextAnswer() {}

    @JsonCreator
    public TextAnswer(Map<String, String> jsonMap) {
        this.id = jsonMap.get("id");
        this.answer = jsonMap.get("answer");
    }

    @JsonValue
    public Map<String, String> toJsonMap() {
        HashMap<String, String> hashMap = new HashMap<>();
        hashMap.put("id", "test");
        hashMap.put("answer", "hello, world!");
        return hashMap;
    }
}

public abstract class MixInTextAnswer {
    @JsonProperty("minInAnswer")
    private String answer;

    /**
     * {@link TextAnswer#toJsonMap()}
     * @return
     */
    @JsonValue(value = false)
    public abstract Map<String, String> toJsonMap();
}

public class TestMain {
    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    public static void main(String[] args) throws IOException {
        OBJECT_MAPPER.addMixIn(TextAnswer.class, MixInTextAnswer.class);
        Test testType = new Test(new TextAnswer());

        String testJsonStr = toJsonString(testType);
        System.out.println(testJsonStr);

        testType = OBJECT_MAPPER.readValue(testJsonStr, Test.class);
        System.out.println(testType.getAnswer().getAnswer());
    }

    private static String toJsonString(Object o) throws JsonProcessingException {
        return OBJECT_MAPPER.writeValueAsString(o);
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    private static class Test{
        private TextAnswer answer;
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    private static class TestType{
        private AnswerType answerType;
    }

    @AllArgsConstructor
    private enum AnswerType {
        TEXT("text", "文本类型答案"), PIC("picture", "图片类型答案");
        private String name;
        private String desc;

        @JsonValue
        public String getType(){
            return name;
        }
    }
}

// output:
// {"answer":{"id":null,"minInAnswer":null}}
// null
```
