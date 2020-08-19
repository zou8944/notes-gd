## 更新

如果需要更新某个JSONB字段中某个字段的值，在使用JOOQ的情况下，可以按照如下方式使用
```kotlin
val newAdditionInfo = JSONB.valueOf(JsonObject().put("category", goods.getString("category")).encode())
set(GOODS.ADDITIONAL_INFO, DSL.field("COALESCE(additional_info, '{}')::jsonb || (?)::jsonb", JSONB::class.java, newAdditionInfo))
```
使用到如下几个点
- JOOQ的DSL.field(sql段, 列类型, 绑定值)；重点在绑定值，有了绑定值就可以使用bindValues获取到对应的参数
- 绑定值的类型一定要是JSONB类型，否则会报错，排查半天也不一定知道是为什么
- COALESCE()一定要，如果初始additional_info为null，则两个jsonb结合的结果也会是null。
