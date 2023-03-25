# Spring Expression Language

Spring Expression Language(SpEL) 是一种强大表达式语言，支持运行时对对象的查询与操作。

SpEL 并不直接绑定到 Spring，可以独立使用。本章中的示例都将 SpEL 用作一种独立的表达式语言。 这需要创建一些引导基础设施类，例如解析器等。 
(大多数 Spring 用户不需要处理这个基础设施，而只需要编写用于评估的表达式字符串。)


定义一个全局的 SpEL 解析器
```java
ExpressionParser parser = new SpelExpressionParser();
```

## Literal expressions

```java
//1.string
String string = (String) parser.parseExpression("'Hello World'").getValue();

//2.1 integer
Integer integer = (Integer) parser.parseExpression("123").getValue();

//2.2 float
Float aFloat = (Float) parser.parseExpression("123.3F").getValue();

//2.3 double
Double aDouble = (Double) parser.parseExpression("1234.45D").getValue();

//3.boolean
Boolean aTrue = (Boolean) parser.parseExpression("true").getValue();

Boolean aFalse = (Boolean) parser.parseExpression("false").getValue();

//4.null
Object aNull = parser.parseExpression("null").getValue();

//other
String value = parser.parseExpression("'Hello World!'").getValue(String.class);
```


## Logic expression
```java
Boolean trueAndTrue = spelExpressionParser.parseExpression("true and true").getValue(Boolean.class);

Boolean trueAndFalse = spelExpressionParser.parseExpression("true and false").getValue(Boolean.class);

Boolean falseAndFalse = spelExpressionParser.parseExpression("false and false").getValue(Boolean.class);

Boolean falseOrTrue = spelExpressionParser.parseExpression("false or true").getValue(Boolean.class);
```

## Operator expression
```java
// evaluates to true
boolean trueValue = parser.parseExpression("2 == 2").getValue(Boolean.class);

// evaluates to false
boolean falseValue = parser.parseExpression("2 < -5.0").getValue(Boolean.class);

// evaluates to true
boolean value = parser.parseExpression("'black' < 'block'").getValue(Boolean.class);

Integer sum = spelExpressionParser.parseExpression("1 + 1").getValue(Integer.class);

String fullName = spelExpressionParser.parseExpression("'linuxea' + 'lin'").getValue(String.class);

Integer mod = spelExpressionParser.parseExpression("10 % 3").getValue(Integer.class);

Integer div = spelExpressionParser.parseExpression("10 / 3").getValue(Integer.class);
```


## Property expression
```java
Person person = new Person("linuxea", 39);

//1.root object

String name = parser.parseExpression("name").getValue(person, String.class);

//2.context for reuse the object
EvaluationContext evaluationContext = new StandardEvaluationContext(person);
Integer age = parser.parseExpression("age").getValue(evaluationContext, Integer.class);

@Data
@AllArgsConstructor
public static class Person {
    private String name;
    private Integer age;
}
```

## Assign expression
```java
//build context for reuse
Person person = new Person();
StandardEvaluationContext standardEvaluationContext = new StandardEvaluationContext(person);
// assign value
spelExpressionParser.parseExpression("scores[0]").setValue(standardEvaluationContext, 100);
// get value
Integer[] scores = person.getScores();


@Data
public static class Person {
    private Integer[] scores = new Integer[5];
}
```

## Constructor expression
```java
String value = spelExpressionParser.parseExpression("new String('hello')")
    .getValue(String.class);

Integer length = spelExpressionParser.parseExpression("length").getValue(value, Integer.class);
```

## Elvis expression
```java
Boolean value = spelExpressionParser.parseExpression("1 == 1 ? true : false").getValue(Boolean.class);
```

## Function expression
```java
StandardEvaluationContext standardEvaluationContext = new StandardEvaluationContext();
standardEvaluationContext.registerFunction("reverse", FuncEL.class.getDeclaredMethod("reverse", String.class));

String value = spelExpressionParser.parseExpression("#reverse('hello')")
    .getValue(standardEvaluationContext, String.class);
```

## List expression
```java
List value = parser.parseExpression("{1, 2, 3, 4}").getValue(List.class);
```


## Index expression
```java
String[] members = new String[]{"linuxea", "linuxea2"};
String value = spelExpressionParser.parseExpression("[1]").getValue(members, String.class);
```

## Map expression
```java
Person person = new Person();
StandardEvaluationContext standardEvaluationContext = new StandardEvaluationContext(person);

String name = parser.parseExpression("map['name']").getValue(standardEvaluationContext, String.class);

@Data
public static class Person {

private final Map<String, String> map = new HashMap<>() {{
    put("name", "linuxea");
    put("age", "39");
}};
}
```


## Method expression
```java
Double random = expressionParser.parseExpression("T(java.lang.Math).random()").getValue(Double.class);

String name = "linuxea";
String biggerName = expressionParser.parseExpression("toUpperCase()").getValue(name, String.class);
Integer length = expressionParser.parseExpression("length()").getValue(name, Integer.class);
```


## Property expression
```java
Person person = new Person("linuxea", 39);

//1.root object
String name = parser.parseExpression("name").getValue(person, String.class);

//2.context for reuse the object
EvaluationContext evaluationContext = new StandardEvaluationContext(person);
Integer age = parser.parseExpression("age").getValue(evaluationContext, Integer.class);


  @Data
  @AllArgsConstructor
  public static class Person {

    private String name;
    private Integer age;
  }
```

## This expression
```java
List<Integer> numbers = new ArrayList<>() {
      {
        add(1);
        add(2);
        add(3);
        add(4);
        add(5);
        add(6);
        add(7);
        add(8);
        add(9);
        add(10);
      }
    };
StandardEvaluationContext standardEvaluationContext = new StandardEvaluationContext();
standardEvaluationContext.setVariable("numbers", numbers);
spelExpressionParser.parseExpression("#numbers.?[#this%2==0]")
    .getValue(standardEvaluationContext, List.class)
    .forEach(System.out::println);
```


## Type expression
```java
Class<Integer> value = spelExpressionParser.parseExpression("T(Integer)").getValue(Class.class);

Class<String> string = spelExpressionParser.parseExpression("T(java.lang.String)")
    .getValue(Class.class);
```


## Value expression
```java
@Value("${numbers}")
private Set<Integer> integers;

@Override
public void run(ApplicationArguments args) throws Exception {
    System.out.println(integers);
}
```


## Variable expression
```java
// context variable
StandardEvaluationContext standardEvaluationContext = new StandardEvaluationContext();
standardEvaluationContext.setVariable("hello", "hello world");

String value = spelExpressionParser.parseExpression("#hello")
    .getValue(standardEvaluationContext, String.class);
```


# Summary

本文介绍了 SpEL 的常用表达式，在合理使用的前提下能够减少模板代码开发，增强对对象的获取与操纵能力。
但是同时也需要避免对表达式的滥用，写出过于复杂的表达式，应该进行合理地拆分，达到可拓展，可维护的目的。



## Reference
- [Spring Expression Language (SpEL)](https://docs.spring.io/spring-framework/docs/3.0.x/reference/expressions.html#expressions-beandef)
- [Spring Expression Language Guide](https://www.baeldung.com/spring-expression-language)