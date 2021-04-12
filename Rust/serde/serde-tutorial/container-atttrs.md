## 容器属性

#### 1. `#[serde(rename = "name")]`

使用给定的名称而不是其 Rust 名称对该结构体或枚举进行序列化与反序列化。

允许为序列化及反序列化指定独立的名称。

 - `#[serde(rename(serialize = "ser_name"))]`
 - `#[serde(rename(deserialize = "de_name"))]`
 - `#[serde(rename(serialize = "ser_name", deserialize = "de_name"))]`

#### 2. `#[serde(rename_all = "...")]`

根据给定的大小写约定重命名所有字段(如果这是一个结构)或 variant (如果这是一个枚举)。可能的值有`lowercase`，`UPPERCASE`，`PascalCase`，`camelCase`，`snake_case`，`SCREAMING_SNAKE_CASE`，`kebab-case`，`SCREAMING-KEBAB-CASE`。

允许为序列化及反序列化指定独立的约定。

 - `#[serde(rename_all(serialize = "..."))]`
 - `#[serde(rename_all(deserialize = "..."))]`
 - `#[serde(rename_all(serialize = "...", deserialize = "..."))]`

#### 3. `#[serde(deny_unknown_fields)]`

在遇到未知字段时，在反序列化期间始终会报错。当此属性不存在时，默认情况下，对于诸如 JSON 之类的自描述结构，未知字段将被忽略。

请注意：在外部结构或扁平化的字段上，不支持将这个属性与 [flatten](https://serde.rs/field-attrs.html#flatten) 结合使用。

#### 4. `#[serde(tag = "type")]`

使用带有内部标记的枚举表示形式来表示此枚举。有关此表示形式的详细信息，请参见[枚举表现形式](./枚举表现形式.md)。

#### 5. `#[serde(tag = "t", content = "c")]`

使用相邻 (adjacently) 标记的枚举表示形式来表示此枚举，并为标签和内容使用给定的字段名称。有关此表示形式的详细信息，请参见[枚举表现形式](./枚举表现形式.md)。

#### 6. `#[serde(untagged)]`

使用未被标记的枚举表示形式来表示此枚举。有关此表示形式的详细信息，请参见[枚举表现形式](./枚举表现形式.md)。

#### 7. `#[serde(bound = "T: MyTrait")]`

`Serialize`和`Deserialize`的 Where 语句的实现。它取代了 Serde 推断的特征范围。

允许为序列化及反序列化指定独立的特征范围：

- `#[serde(bound(serialize = "T: MySerTrait"))]`
- `#[serde(bound(deserialize = "T: MyDeTrait"))]`
- `#[serde(bound(serialize = "T: MySerTrait", deserialize = "T: MyDeTrait"))]`

#### 8. `#[serde(default)]`

当反序列化时，任何缺失的字段都会通过结构体实现的`Default`来进行填充。仅支持结构体。

#### 9. `#[serde(default = "path")]`

当反序列化时，任何缺失的字段都会通过对象返回的给定函数或方法进行填充。函数必须形如`fn() -> T`。举个例子：`default = "my_default"`将会调用`my_default()`并且`default = "SomeTrait::some_default"`将会调用`SomeTrait::some_default()`。仅支持结构体。

#### 10. `#[serde(remote = "...")]`

用来为[远程类型](./remote-derive.md)派生`Serialize`和`Deserialize`。

#### 11. `#[serde(transparent)]`

序列化与反序列化一个仅有一个字段的 newtype 结构体 或支撑 (braced) 结构，就好像序列化与反序列化它们的字段本身一样。和`#[repr(transparent)]`类似。

#### 12. `#[serde(from = "FromType")]`

通过反序列化为`FromType`进行反序列化，然后进行转换。这个类型必须实现了`From<FromType>`，并且`FromType`必须实现了`Deserialize`。

#### 13. `#[serde(try_from = "FromType")]`

通过反序列化为`FromType`进行反序列化，然后进行转换，但是可能会出错。这个类型必须实现`TryFrom<FromType>`，并带有实现了`Display`的错误类型。`FromType`必须实现`Deserialize`。

#### 14. `#[serde(into = "IntoType")]`

通过将这个类型转化为指定的`IntoType`并对其进行序列化，从而对这个类型进行序列化。这个类型必须实现`Clone`和`Into<IntoType>`，并且`IntoType`必须实现`Serialize`。

#### 15. `#[serde(crate = "...")]`

指定从生成的代码中引用 Serde API 时要使用的 Serde crate 实例的路径。这通常仅适用于从不同的 crate 中的公共宏中调用重新导出的 Serde 派生。