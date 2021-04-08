## 容器属性

 `#[serde(rename = "name")]`

使用给定的名称而不是其 Rust 名称对该结构体或枚举进行序列化与反序列化。

允许为序列化及反序列化指定独立的名称。

  - `#[serde(rename(serialize = "ser_name"))]`
  - `#[serde(rename(deserialize = "de_name"))]`
  - `#[serde(rename(serialize = "ser_name", deserialize = "de_name"))]`

  