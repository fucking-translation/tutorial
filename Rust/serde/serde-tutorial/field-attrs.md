## 字段属性

#### 1. `#[serde(rename = "name")]`

使用给定的名称而不是其 Rust 名称对该字段进行序列化与反序列化。这对于[将字段序列化成驼峰命名格式](./attr-rename.md)或序列化保留的 Rust 关键字很有用。

允许为序列化及反序列化指定独立的名称。

 - `#[serde(rename(serialize = "ser_name"))]`
 - `#[serde(rename(deserialize = "de_name"))]`
 - `#[serde(rename(serialize = "ser_name", deserialize = "de_name"))]`

 #### 2. `#[serde(alias = "name")]`

通过给定的名称或者其 Rust 名称对字段进行反序列化。同一个字段可能会重复的指定多个可能的名称。

#### 3. `#[serde(default)]`

如果当序列化时该字段没有值，则使用`Default::default()`对其进行填充。

#### 4. `#[serde(default = "path")]`

当反序列化时，任何缺失的字段都会通过给定的函数获取默认值。函数必须形如`fn() -> T`。举个例子：`default = "empty_value"`将会调用`empty_value()`并且`default = "SomeTrait::some_default"`将会调用`SomeTrait::some_default()`。

#### 5. `#[serde(flatten)]`

将此字段的内容扁平化到定义它的容器中。

这消除了序列化表示及 Rust 数据结构表示之间的一级结构。它可用于将公共键拆分为共享结构，或者使用任意字符串将剩余字段捕获到映射中。[结构扁平化](./attr-flatten.md)提供了一些示例。

请注意：这个属性不支持与带有 [deny_unknown_field](./container-attrs.md#deny_unknown_fields) 的结构体结合使用。外部和内部扁平化结构均不应使用此属性。

#### 6. `#[serde(skip)]`

跳过此字段，不对其进行序列化及反序列化。

#### 7. `#[serde(skip_serializing)]`

当序列化时跳过此字段，但依然对其反序列化。

#### 8. `#[serde(skip_deserializing)]`

当反序列化时跳过此字段，但依然对其序列化。

#### 9. `#[serde(skip_serializing_if = "path")]`

调用函数来决定是否对该字段进行序列化。给定的函数必须形如`fn(&T) -> bool`，举个例子，`skip_serializing_if = "Option::is_none"`将会跳过一个为 None 的 Option。

#### 10. `#[serde(serialize_with = "path")]`

使用与其`Serialize`实现不同的函数来序列化此字段。给定的函数必须形如`fn<S>(&T, S) -> Result<S::Ok, S::Error> where S: Serializer`。使用`serialize_with`的字段不需要能够派生`Serialize`。

#### 11. `#[serde(deserialize_with = "path")]`

使用与其`Deserialize`实现不同的函数来反序列化此字段。给定的参数必须形如`fn<'de, D>(D) -> Result<T, D::Error> where D: Deserializer<'de>`。使用`deserialize_with`的字段不需要能够派生`Deserialize`。

#### 12. `#[serde(with = "module")]`

将`serialize_with`和`deserialize_with`结合使用。Serde 将会使用`$module::serialize`作为`serialize_with`的函数，使用`$module::deserialize`作为`deserialize_with`的函数。

#### 13. `#[serde(borrow)]`和`#[serde(borrow = "'a + 'b + ...")]`

从使用零拷贝反序列化的反序列化器中为该字段借用数据。请参阅[这个示例](./lifetimes.md#borrowing-data-in-a-derived-impl)。

#### 14. `#[serde(bound = "T: MyTrait")]`

`Serialize`和`Deserialize`的 Where 语句的实现。它取代了 Serde 为当前字段推断的特征范围。

允许为序列化及反序列化指定独立的特征范围：

- `#[serde(bound(serialize = "T: MySerTrait"))]`
- `#[serde(bound(deserialize = "T: MyDeTrait"))]`
- `#[serde(bound(serialize = "T: MySerTrait", deserialize = "T: MyDeTrait"))]`

#### 15. `#[serde(getter = "...")]`

当为具有一个或多个私有字段的[远程类型](./remote-derive.md)派生`Serialize`时使用。