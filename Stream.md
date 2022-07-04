## Stream

#### 对List<Object>中Object的某个字段进行求和

```java
BigDecimal salesAmount = order.stream().map(Order::getPayPrice).reduce(BigDecimal.ZERO, BigDecimal::add);
```

#### 取List<Object>中Object的某个字段放到新的list中

```java
List<Integer> userIds = order.stream().map(Order::getId).collect(Collectors.toList());
```

#### 将list转成map

```java
// date做key，data为value
Map<String, Integer> dateMap = dataList.stream()
                .collect(Collectors.toMap(TrendVO::getDate, TrendVO::getData));
```

