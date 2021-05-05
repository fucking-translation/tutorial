## 编写一个数据格式

在编写一个数据格式之前要理解的一个重要的事情就是 **Serde 不是一个解析库**。Serde 的所有内容都不会帮助你解析所要实现的任何格式。Serde 的角色十分特别：

- **序列化** - 获取用户任意的 (arbitrary) 数据结构并以最高的效率将其转化为某个格式。
- **反序列化** - 以最高的效率将你解析的数据解释成用户选择的数据结构。

解析不是这些事情，你要么从头开始编写解析代码，要么使用解析库来实现你的反序列化器。

第二重要的事情是要理解 [Serde 数据模型](./data-model.md)。

以下页面介绍了使用 Serde 实现的基本但功能正常的 JSON 序列化器和反序列化器。

- [在 crate 根路径要导出什么约定](./conventions.md)
- [Serde 错误特征和错误处理](./error-handling.md)
- [实现一个序列化器](./impl-serializer.md)
- [实现一个反序列化器](./impl-deserializer.md)

你可以在 [Github](https://github.com/serde-rs/example-format) 仓库中找到这四个可构建的源文件。