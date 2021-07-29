---
title: "Jackson 多态类型解决方案"
# subtitle: "Java Jackson 多态类型解决方案"
layout: post
author: "Wentao Dong"
date: 2021-07-18 21:00:00
catalog: true
header-style: post
header-img: "img/in-post/city_night.png"
tags:
  - Java
  - Jackson
  - 多态
---

#### Jackson 版本
- 2.6.7

#### 多态类型常用注解

- **@JsonTypeInfo**：注解到基类/接口上，用于定义如何序列化和反序列化类型信息

- **@JsonSubTypes**：常与JsonTypeInfo配合使用（不是必须的，如果可以确定类型就不需要，比如use=JsonTypeInfo.Id.CLASS），用于列举子类及对应的类型识别码（类型识别码也可以直接在子类上使用@JsonTypeName定义）

#### **@JsonTypeInfo** 属性**use**: 用于定义类型识别码
- JsonTypeInfo.Id.NONE：不使用类型识别码。此种状态下，类可以正常序列化，但是不产生类型识别码，不能直接反序列化

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NONE, defaultImpl = TextAnswer.class)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class), @JsonSubTypes.Type(value = AudioAnswer.class)})
public abstract class Answer {
    private String id;
}
// Test test = new Test(new AudioAnswer());
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":{"url":null,"filename":null,"length":null,"text":null}}
```

- JsonTypeInfo.Id.CLASS：使用类的全限定名作为类型识别码，默认识别码名称：@class

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS, defaultImpl = TextAnswer.class)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class), @JsonSubTypes.Type(value = AudioAnswer.class)})
public abstract class Answer {
    private String id;
}
// Test test = new Test(new AudioAnswer());
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":{"@class":"org.json.test.AudioAnswer","url":null,"filename":null,"length":null,"text":null}}
```

- JsonTypeInfo.Id.MINIMAL_CLASS：使用相对基类路径作为类型识别码，默认识别码名称：@c。此状态下，类型识别码以"."开头。注意如果对象与基类不在同一个文件夹下，直接序列化该对象，产生的识别码是有问题的，需要再包装一层（如下面的Test类包Answer类）。可以修改下面的代码验证一下

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.MINIMAL_CLASS, defaultImpl = TextAnswer.class)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class), @JsonSubTypes.Type(value = AudioAnswer.class)})
public abstract class Answer {
    private String id;
}
// Test test = new Test(new AudioAnswer());
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":{"@c":".AudioAnswer","url":null,"filename":null,"length":null,"text":null}}
```

- JsonTypeInfo.Id.NAME：使用指定的名字作为类型识别码，默认识别码为类名，默认识别码名称：@type。与@JsonSubTypes 配合使用。

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, defaultImpl = TextAnswer.class)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class, name = "ta"), @JsonSubTypes.Type(value = AudioAnswer.class, name = "aa")})
public abstract class Answer {
    private String id;
}
// Test test = new Test(new AudioAnswer());
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":{"@type":"aa","url":null,"filename":null,"length":null,"text":null}}
```

- JsonTypeInfo.Id.CUSTOM：用户自定义类型识别码，与@JsonTypeResolver 配合使用，需要自定义类型解析器（不常用）。以下例子实现了一个输出类型识别码时给类型识别码增加"@test" 后缀的功能。

```java
public class CustomTypeResolverBuilder extends StdTypeResolverBuilder {
    /**
     * Helper method that will either return configured custom
     * type id resolver, or construct a standard resolver
     * given configuration.
     */
    protected TypeIdResolver idResolver(MapperConfig<?> config,
                                        JavaType baseType, Collection<NamedType> subtypes, boolean forSer, boolean forDeser)
    {
        // Custom id resolver?
        if (_customIdResolver != null) { return _customIdResolver; }
        if (_idType == null) throw new IllegalStateException("Can not build, 'init()' not yet called");
        switch (_idType) {
            case CUSTOM: // need custom resolver...
                return new CustomNameIdResolver(baseType, config.getTypeFactory());
            default:
                return super.idResolver(config, baseType, subtypes, forDeser, forDeser);
        }
    }
}

public class CustomNameIdResolver extends ClassNameIdResolver {
    private static final String ID_SUFFIX = "@test";
    public CustomNameIdResolver(JavaType baseType, TypeFactory typeFactory) {
        super(baseType, typeFactory);
    }

    @Override
    public String idFromValue(Object value) {
        String idFrom = _idFrom(value, value.getClass());
        return idFrom + ID_SUFFIX;
    }

    @Override
    public String idFromValueAndType(Object value, Class<?> type) {
        return _idFrom(value, type);
    }

    @Deprecated // since 2.3
    @Override
    public JavaType typeFromId(String id) {
        return _typeFromId(id, _typeFactory);
    }

    @Override
    public JavaType typeFromId(DatabindContext context, String id) {
        int endIdx = id.indexOf(ID_SUFFIX);
        if (endIdx != -1) {
            id = id.substring(0, endIdx);
        }
        return _typeFromId(id, context.getTypeFactory());
    }

    @Override
    public JsonTypeInfo.Id getMechanism() {
        return JsonTypeInfo.Id.CUSTOM;
    }
}

@JsonTypeInfo(use = JsonTypeInfo.Id.CUSTOM, property = "className", defaultImpl = TextAnswer.class)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class, name = "ta"), @JsonSubTypes.Type(value = AudioAnswer.class, name = "aa")})
@JsonTypeResolver(CustomTypeResolverBuilder.class)
public abstract class Answer {
    private String id;
}
// Test test = new Test(new AudioAnswer());
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":{"className":"org.json.test.AudioAnswer@test","url":null,"filename":null,"length":null,"text":null}}
```

#### **@JsonTypeInfo** 属性**property**: 用于指定识别码的名称

- 默认值，使用上述定义识别码时的默认识别码名称; json序列化时会使用这个做为key, 识别码做为value。即{识别码名称: 识别码}

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "className", defaultImpl = TextAnswer.class)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class, name = "ta"), @JsonSubTypes.Type(value = AudioAnswer.class, name = "aa")})
public abstract class Answer {
    private String id;
}
// Test test = new Test(new AudioAnswer());
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":{"className":"aa","url":null,"filename":null,"length":null,"text":null}}
```

#### **@JsonTypeInfo** 属性**include**: 用于定义如何将<识别码名称, 识别码> 放到json 序列化出的字符串里
- PROPERTY：默认值，将<识别码名称, 识别码>做为对象的一个属性，形式为{识别码名称:识别码, 对象的其他属性}

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
private static class Test{
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "className", defaultImpl = TextAnswer.class)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class), @JsonSubTypes.Type(value = AudioAnswer.class)})
    private Answer answer;
}

// Test test = new Test(new AudioAnswer());
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":{"className":"AudioAnswer","url":null,"filename":null,"length":null,"text":null}}
```

- WRAPPER_OBJECT: 在对象外面再包一层，包装对象，形式为{识别码: 对象}

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
private static class Test{
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "className", defaultImpl = TextAnswer.class, include = JsonTypeInfo.As.WRAPPER_OBJECT)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class), @JsonSubTypes.Type(value = AudioAnswer.class)})
    private Answer answer;
}

// Test test = new Test(new AudioAnswer());
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":{"AudioAnswer":{"url":null,"filename":null,"length":null,"text":null}}}
```

- WRAPPER_ARRAY: 也是在对象外面再包一层，包成数组，形式为[识别码, 对象]

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
private static class Test{
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "className", defaultImpl = TextAnswer.class, include = JsonTypeInfo.As.WRAPPER_ARRAY)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class), @JsonSubTypes.Type(value = AudioAnswer.class)})
    private Answer answer;
}

// Test test = new Test(new AudioAnswer());
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":["AudioAnswer",{"url":null,"filename":null,"length":null,"text":null}]}
```

- EXTERNAL_PROPERTY: 仅当在属性上注解时有效，在对象上注解行为会与PROPERTY一致；做为对象的外部属性，形式为{属性名: 对象, 识别码名称:识别码,}，

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
private static class Test{
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "className", defaultImpl = TextAnswer.class, include = JsonTypeInfo.As.EXTERNAL_PROPERTY)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class), @JsonSubTypes.Type(value = AudioAnswer.class)})
    private Answer answer;
}

// Test test = new Test(new AudioAnswer());
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":{"url":null,"filename":null,"length":null,"text":null},"className":"AudioAnswer"}
```

- EXISTING_PROPERTY: 使用已经存在的属性的值做为类型识别码，这意味着序列化时我们必须自己设置该值，反序列化时行为与PROPERTY一致，效果上看与在属性上加@JsonTypeId 差不多。EXISTING_PROPERTY是不生成类型标识符，而在属性上加@JsonTypeId 是覆盖生成的标识符，效果都是使用了自己手动指定的类型标识符

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
private static class Test{
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "className", defaultImpl = TextAnswer.class, include = JsonTypeInfo.As.EXISTING_PROPERTY)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class, name = "ta"), @JsonSubTypes.Type(value = AudioAnswer.class, name = "aa")})
    private Answer answer;
}

// AudioAnswer audioAnswer = new AudioAnswer();
// audioAnswer.setClassName("aa");
// Test test = new Test(audioAnswer);
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":{"className":"aa","url":null,"filename":null,"length":null,"text":null}}

@Data
@NoArgsConstructor
@AllArgsConstructor
private static class Test{
    @JsonTypeInfo(use = JsonTypeInfo.Id.NAME, defaultImpl = TextAnswer.class)
    @JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class, name = "ta"), @JsonSubTypes.Type(value = AudioAnswer.class, name = "aa")})
    private Answer answer;
}

public abstract class Answer {
    private String id;
    @JsonTypeId
    private String className;

    public void setClassName(String className){
        this.className = className;
    }

    public String getClassName(){
        return className;
    }
}

// AudioAnswer audioAnswer = new AudioAnswer();
// audioAnswer.setClassName("aa");
// Test test = new Test(audioAnswer);
// String testJsonStr = toJsonString(test);
// System.out.println(testJsonStr);
// output: 
// {"answer":{"@type":"aa","url":null,"filename":null,"length":null,"text":null}}
```

#### **@JsonTypeInfo** 属性**defaultImpl**: 用于指定默认实现
- 用于类别识别码的值缺失或者根据标识码的值没有找到对应的类时的默认类型。如果没有定义默认类型，则当前述情况发生时，将抛出异常

#### **@JsonTypeInfo** 属性**visible**: 用于指定类型识别码可见性
- 类型识别码信息<识别码名称, 识别码> 在反序列化对象时是否可见，默认是不可见的（仅用于确定类型），Jackson将在反序列化对象前删除类型识别码信息

#### Talk is cheap, show me your code

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "className", defaultImpl = TextAnswer.class)
@JsonSubTypes(value = {@JsonSubTypes.Type(value = TextAnswer.class), @JsonSubTypes.Type(value = AudioAnswer.class)})
public abstract class Answer {
    private String id;
}
```

```java
@Data
public class AudioAnswer extends Answer {
    protected String url;
    private String filename;
    private Integer length;
    private String text;
}
```

```java
@Data
public class TextAnswer extends Answer {
    private String answer;
}
```

```java
public class TestMain {
    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    public static void main(String[] args) throws IOException {
        Test test = new Test(new AudioAnswer());

        String testJsonStr = toJsonString(test);
        System.out.println(testJsonStr);

        test = OBJECT_MAPPER.readValue(testJsonStr, Test.class);
        System.out.println(test.getAnswer());
    }

    private static String toJsonString(Object o) throws JsonProcessingException {
        return OBJECT_MAPPER.writeValueAsString(o);
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    private static class Test{
        private Answer answer;
    }
}
```
注意：上述代码中某些属性和方法未列出，如：className, setClassName, getClassName
纸上得来终觉浅，绝知此事要躬行。前面说的有不理解的，可以动手改改，跑一跑代码，加深理解