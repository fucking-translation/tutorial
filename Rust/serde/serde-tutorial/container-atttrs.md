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