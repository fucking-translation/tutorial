## 变体 (Variant) 属性

#### 1. `#[serde(rename = "name")]`

使用给定的名称而不是其 Rust 名称对该变体进行序列化与反序列化。

允许为序列化及反序列化指定独立的名称。

 - `#[serde(rename(serialize = "ser_name"))]`
 - `#[serde(rename(deserialize = "de_name"))]`
 - `#[serde(rename(serialize = "ser_name", deserialize = "de_name"))]`

#### 2. `#[serde(alias = "name")]`

通过给定的名称或者其 Rust 名称对变体进行反序列化。同一个变体可能会重复的指定多个可能的名称。

#### 3. `#[serde(rename_all = "...")]`

根据给定的大小写约定重命名所有字段(如果这是一个结构)或 variant (如果这是一个枚举)。可能的值有`lowercase`，`UPPERCASE`，`PascalCase`，`camelCase`，`snake_case`，`SCREAMING_SNAKE_CASE`，`kebab-case`，`SCREAMING-KEBAB-CASE`。

允许为序列化及反序列化指定独立的约定。

 - `#[serde(rename_all(serialize = "..."))]`
 - `#[serde(rename_all(deserialize = "..."))]`
 - `#[serde(rename_all(serialize = "...", deserialize = "..."))]`

#### 4. `#[serde(skip)]`

永远不会对该变体进行序列化与反序列化操作。

##### 5. `#[serde(skip_serializing)]`

永远不会对该变体进行序列化。尝试序列化此变体会视为错误。


#### 6. `#[serde(skip_deserializing)]`

永远不会对该变体进行反序列化。

#### 7. `#[serde(serialize_with = "path")]`

使用与其`Serialize`实现不同的函数来序列化此变体。给定的函数必须形如`fn<S>(&FIELD0, &FIELD1, ..., S) -> Result<S::Ok, S::Error> where S: Serializer`。使用`serialize_with`的变体不需要能够派生`Serialize`。

`FIELD{n}`对于变体的每个字段都存在。因此一个单元变体只有一个`S`参数，并且元组/结构体变体的每一个字段都会有一个参数。

#### 8. `#[serde(deserialize_with = "path")]`

使用与其`Deserialize`实现不同的函数来反序列化此变体。给定的参数必须形如`fn<'de, D>(D) -> Result<FIELDS, D::Error> where D: Deserializer<'de>`。使用`deserialize_with`的变体不需要能够派生`Deserialize`。

`FIELDS`是变体所有字段的元组。一个单元变体将会使用`()`作为它的`FIELDS`类型。

#### 9. `#[serde(with = "module")]`

将`serialize_with`和`deserialize_with`进行结合。Serde 将会使用`$module::serialize`作为`serialize_with`的函数，使用`$module::deserialize`作为`deserialize_with`的函数。

#### 10. `#[serde(bound = "T: MyTrait")]`

`Serialize`和`Deserialize`的 Where 语句的实现。它取代了 Serde 为当前变体推断的特征范围。

允许为序列化及反序列化指定独立的特征范围：

- `#[serde(bound(serialize = "T: MySerTrait"))]`
- `#[serde(bound(deserialize = "T: MyDeTrait"))]`
- `#[serde(bound(serialize = "T: MySerTrait", deserialize = "T: MyDeTrait"))]`

#### 11. `#[serde(borrow)]`和`#[serde(borrow = "'a + 'b + ...")]`

从使用零拷贝反序列化的反序列化器中为该字段借用数据。请参阅[这个示例](https://serde.rs/lifetimes.html#borrowing-data-in-a-derived-impl)。只允许在 newtype 变体上 (只有一个字段的元组辩题) 使用此属性。

#### 12. `#[serde(other)]`

如果枚举标签不是此枚举中其他变体之一的标签，则反序列化此变体。仅允许在内部标记或相邻标记的枚举内部的单元变体上使用。

举个例子：如果我们有一个带有`serde(tag = "variant")`的内部标记的枚举，该枚举包含`A`，`B`即标记了`serde(other)`的`Unknown`变体，只要输入的`“variant”`字段既不是`“A”`也不是`“B”`则将其序列化。