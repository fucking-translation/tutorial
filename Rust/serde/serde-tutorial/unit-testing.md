# 单元测试

[serde_test](https://docs.serde.rs/serde_test/) 提供了一个方便简洁的 (concise) 方式为`Serialize`和`Deserialize`编写单元测试。

值的序列化可以通过在序列化值的过程中得到的[Serializer](https://docs.serde.rs/serde/ser/trait.Serializer.html)顺序调用进行表征，因此`serde_test`提供了一个 [Token](https://docs.serde.rs/serde_test/enum.Token.html) 抽象，大致对应于`Serializer`方法调用。它提供了一个`assert_ser_tokens`函数测试一个值序列化为特定的方法调用序列，`assert_de_tokens`函数测试一个值可以从特定的方法调用序列中反序列化出来，以及`assert_tokens`函数同时可以测试这两个功能。它还提供了测试预期错误条件的功能。

在 [linked-hash-map](https://github.com/contain-rs/linked-hash-map) crate 中有一个示例。

```rust
use linked_hash_map::LinkedHashMap;
use serde_test::{Token, assert_tokens};

#[test]
fn test_ser_de_empty() {
    let map = LinkedHashMap::<char, u32>::new();

    assert_tokens(&map, &[
        Token::Map { len: Some(0) },
        Token::MapEnd,
    ]);
}

#[test]
fn test_ser_de() {
    let mut map = LinkedHashMap::new();
    map.insert('b', 20);
    map.insert('a', 10);
    map.insert('c', 30);

    assert_tokens(&map, &[
        Token::Map { len: Some(3) },
        Token::Char('b'),
        Token::I32(20),

        Token::Char('a'),
        Token::I32(10),

        Token::Char('c'),
        Token::I32(30),
        Token::MapEnd,
    ]);
}
```