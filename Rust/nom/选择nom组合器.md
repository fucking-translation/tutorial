# nom 宏解析器和组合器的清单

**请注意**：此清单旨在提供一种比查阅 docs.rs 文档更好的查询 nom 宏的方式，因为 rustdoc 将所有的宏都放在最顶层。功能组合器组织在模块中，因此比较容易找到。

## 基本元素

这些在你的语法中被用于识别底层元素，如：“这是一个点”或“这是一个大端整数”。

| 组合器 | 用法 | 输入 | 输出 | 说明 |
|---|---|---|---|---|
| [char](https://docs.rs/nom/latest/nom/macro.char.html) | `char!('a')` |  `"abc"` | `Ok(("bc", 'a'))` |匹配一个字符(也适用于非 ASCII 字符) | 
| [is_a](https://docs.rs/nom/latest/nom/macro.is_a.html) | ` is_a!("ab")` |  `"ababc"` | `Ok(("c", "abab"))` |匹配作为参数传递的任何字符序列|
| [is_not](https://docs.rs/nom/latest/nom/macro.is_not.html) | `is_not!("cd")` |  `"ababc"` | `Ok(("c", "abab"))` |匹配不包含任何作为参数传递的字符序列|
| [one_of](https://docs.rs/nom/latest/nom/macro.one_of.html) | `one_of!("abc")` |  `"abc"` | `Ok(("bc", 'a'))` |匹配提供的字符之一(也适用于非 ASCII 字符)|
| [none_of](https://docs.rs/nom/latest/nom/macro.none_of.html) | `none_of!("abc")` |  `"xyab"` | `Ok(("yab", 'x'))` |匹配提供字符以外的任何字符|
| [tag](https://docs.rs/nom/latest/nom/macro.tag.html) | `tag!("hello")` |  `"hello world"` | `Ok((" world", "hello"))` |识别特定的字符集或字节|
| [tag_no_case](https://docs.rs/nom/latest/nom/macro.tag_no_case.html) | `tag_no_case!("hello")` |  `"HeLLo World"` | `Ok((" World", "HeLLo"))` |大小写不敏感的比较。请注意大小写不敏感比较的定义对于 unicode 来说不是很明确，所以得到的结果可能和你预期的有出入|
| [take](https://docs.rs/nom/latest/nom/macro.take.html) | `take!(4)` |  `"hello"` | `Ok(("o", "hell"))` |接受特定数量的字节或字符|
| [take_while](https://docs.rs/nom/latest/nom/macro.take_while.html) | `take_while!(is_alphabetic)` |  `"abc123"` | `Ok(("123", "abc"))` |当提供的函数返回 true 时，返回满足条件的最长字节数组。`take_while1`同理，但是必须至少返回一个字符|
| [take_till](https://docs.rs/nom/latest/nom/macro.take_till.html) | `take_till!(is_alphabetic)` |  `"123abc"` | `Ok(("abc", "123"))` |返回最长的字节数组或字符串，直到提供的函数返回 true。`take_till1`同理，但是必须至少返回一个字符。这与`take_while`的行为相反：`take_till!(f)`等价于`take_while!(\|c\| !f(c))`|
| [take_until](https://docs.rs/nom/latest/nom/macro.take_until.html) | `take_until!("world")` |  `"Hello world"` | `Ok(("world", "Hello "))` |返回最长的字节数组或字符串，直到发现提供的标识。`take_until1`同理，但是必须至少返回一个字符。|

## 选择组合器

| 组合器 | 用法 | 输入 | 输出 | 说明 |
|---|---|---|---|---|
| [alt](https://docs.rs/nom/latest/nom/macro.alt.html) | `alt!(tag!("ab") \| tag!("cd"))` |  `"cdef"` | `Ok(("ef", "cd"))` |尝试执行一组解析器，并返回第一个解析成功的结果|
| [switch](https://docs.rs/nom/latest/nom/macro.switch.html) | `switch!(take!(2), "ab" => tag!("XYZ") \| "cd" => tag!("123"))` | `"cd1234"` | `Ok(("4", "123"))` |如果第一个解析器解析成功，则根据其解析结果选择下一个解析器，并返回第二个解析器解析的结果|
| [permutation](https://docs.rs/nom/latest/nom/macro.permutation.html) | `permutation!(tag!("ab"), tag!("cd"), tag!("12"))` | `"cd12abc"` | `Ok(("c", ("ab", "cd", "12"))` |如果所有的子解析器全部执行成功则表示其解析成功，不管子解析器是什么解析顺序|

## 序列组合器

| 组合器 | 用法 | 输入 | 输出 | 说明 |
|---|---|---|---|---|
| [delimited](https://docs.rs/nom/latest/nom/macro.delimited.html) | `delimited!(char!('('), take!(2), char!(')'))` | `"(ab)cd"` | `Ok(("cd", "ab"))` ||
| [preceded](https://docs.rs/nom/latest/nom/macro.preceded.html) | `preceded!(tag!("ab"), tag!("XY"))` | `"abXYZ"` | `Ok(("Z", "XY"))` ||
| [terminated](https://docs.rs/nom/latest/nom/macro.terminated.html) | `terminated!(tag!("ab"), tag!("XY"))` | `"abXYZ"` | `Ok(("Z", "ab"))` |获取第一个解析器解析的值，然后在第二个解析器中进行匹配，并将匹配的结果丢弃|
| [pair](https://docs.rs/nom/latest/nom/macro.pair.html) | `pair!(tag!("ab"), tag!("XY"))` | `"abXYZ"` | `Ok(("Z", ("ab", "XY")))` ||
| [separated_pair](https://docs.rs/nom/latest/nom/macro.separated_pair.html) | `separated_pair!(tag!("hello"), char!(','), tag!("world"))` | `"hello,world!"` | `Ok(("!", ("hello", "world")))` |获取第一个解析器解析的值，然后匹配中间的解析器，并将结果丢弃，然后获取第二个解析器解析的值|
| [tuple](https://docs.rs/nom/latest/nom/macro.tuple.html) | `tuple!(tag!("ab"), tag!("XY"), take!(1))` | `"abXYZ!"` | `Ok(("!", ("ab", "XY", "Z")))` |链接解析器并将子解析器的结果组合到一个元组中。你可以在元组中放置尽可能多的子解析器|
| [do_parse](https://docs.rs/nom/latest/nom/macro.do_parse.html) | `do_parse!(tag: take!(2) >> length: be_u8 >> data: take!(length) >> (Buffer { tag: tag, data: data}) )` | `&[0, 0, 3, 1, 2, 3][..]` | `Buffer { tag: &[0, 0][..], data: &[1, 2, 3][..] }` |`do_parse`依次执行子解析器。它可以存储中间结果并提供给以后的解析器|

## 多次执行一个解析器

| 组合器 | 用法 | 输入 | 输出 | 说明 |
|---|---|---|---|---|
| [count](https://docs.rs/nom/latest/nom/macro.count.html) | `count!(take!(2), 3)` | `"abcdefgh"` | `Ok(("gh", vec!("ab", "cd", "ef")))` |将子解析器执行指定的次数|
| [many0](https://docs.rs/nom/latest/nom/macro.many0.html) | `many0!(tag!("ab"))` |  `"abababc"` | `Ok(("c", vec!("ab", "ab", "ab")))` |将解析器执行 0 次或多次，并在 Vec 中返回结果列表。`many1`同理，但是必须至少返回一个元素|
| [many_m_n](https://docs.rs/nom/latest/nom/macro.many_m_n.html) | `many_m_n!(1, 3, tag!("ab"))` | `"ababc"` | `Ok(("c", vec!("ab", "ab")))` |将解析器执行 m 到 n 次(包括 n)，并在 Vec 中返回结果列表|
| [many_till](https://docs.rs/nom/latest/nom/macro.many_till.html) | `many_till!(tag!( "ab" ), tag!( "ef" ))` | `"ababefg"` | `Ok(("g", (vec!("ab", "ab"), "ef")))` |Applies the first parser until the second applies. 返回一个包含第一个解析器在 Vec 中的结果列表以及第二个解析器的解析结果|
| [separated_list](https://docs.rs/nom/latest/nom/macro.separated_list.html) | `separated_list!(tag!(","), tag!("ab"))` | `"ab,ab,ab."` | `Ok((".", vec!("ab", "ab", "ab")))` |`separated_nonempty_list`同`separated_list`一样，但是必须至少返回一个元素|
| [fold_many0](https://docs.rs/nom/latest/nom/macro.fold_many0.html) | `fold_many0!(be_u8, 0, \|acc, item\| acc + item)` | `[1, 2, 3]` | `Ok(([], 6))` |将解析器执行 0 次或多次，并将返回的结果列表进行聚合(折叠)。`fold_many1`必须至少让子解析器执行一次|
| [fold_many_m_n](https://docs.rs/nom/latest/nom/macro.fold_many_m_n.html) | `fold_many_m_n!(1, 2, be_u8, 0, \|acc, item\| acc + item)` | `[1, 2, 3]` | `Ok(([3], 3))` |将解析器执行 m 到 n 次(包括 n) 并聚合(折叠)返回的结果列表|
| [length_count](https://docs.rs/nom/latest/nom/macro.length_count.html) | `length_count!(number, tag!("ab"))` | `"2ababab"` | `Ok(("ab", vec!("ab", "ab")))` |从第一个解析器中获取一个数字，并多次执行第二个解析器|

## 整型

从二进制格式中解析整型有两种方式：使用解析函数，或者具有可配置字节序的组合器：

- **可配置字节序：** `i16!`，`i32!`，`i64!`, `u16!`，`u32!`，`u64!`是将`nom::Endianness`作为参数的组合器，如：`i16!(endianness)`。如果参数是`nom::Endianness::Big`，则解析大端`i16`整型，否则是小端`i16`整型。
- **固定字节序：**: 对于大端数字，函数以`be_`为前缀，对于小端数字，函数以`le_`为前缀，并且后缀是它们要解析的类型。举个例子：`be_u32`用于解析一个存储在32位中的大端无符号整型。
- `be_f32`，`be_f64`，`le_f32`，`le_f64`：识别浮点数
- `be_i8`，`be_i16`，`be_i24`，`be_i32`，`be_i64`：大端有符号整型
- `be_u8`，`be_u16`，`be_u24`，`be_u32`，`be_u64`：大端无符号整型
- `le_i8`，`le_i16`，`le_i24`，`le_i32`，`le_i64`：小端有符号整型
- `le_u8`，`le_u16`，`le_u24`，`le_u32`，`le_u64`：小端无符号整型

## 流相关

- `eof!`: 如果在输入数据的结尾，则返回该输入
- `complete!`: 取代解析失败的子解析器返回的`Incomplete`

## 修饰符

- `cond!`: 条件组合器。封装其他的解析器并当条件满足使对其进行调用
- `flat_map!`: 
- `map!`: 将函数映射到解析器的结果
- `map_opt!`: 将返回`Option`的函数映射到解析器的输出
- `map_res!`: 将返回`Result`的函数映射到解析器的输出
- `not!`: 仅当嵌入的解析器返回`Error`或`Incomplete`时返回结果。不消耗输入
- `opt!`: 使基础解析器成为可选
- `opt_res!`: 使基础解析器成为可选
- `parse_to!`: 使用来自`std::str::FromStr`的parse函数将当前的输入转换成指定的类型
- `peek!`: 返回解析结果但不消耗输入
- `recognize!`: 如果子解析器解析成功，将消耗的输入作为结果返回
- `consumed()`: 如果子解析器解析成功，将消耗的输入以及产生的输出作为元组返回
- `return_error!`: 当子解析器解析失败时，防止回溯
- `tap!`: 允许在不影响解析器结果的前提下访问该结果
- `verify!`: 如果满足校验函数，则返回子解析器的解析结果

## 错误管理和调试

- `add_return_error!`: 如果子解析器解析失败则添加一个错误
- `dbg!`: 如果解析失败则打印一些信息
- `dbg_dmp!`: 如果解析失败则打印输入和一些信息
- `error_node_position!`: 根据`nom::ErrorKind`以及输入中报错的位置和解析树中的下一个错误创建一个解析错误。如果`verbose-errors`属性没有激活，默认只打印错误代码。
- `error_position!`: 根据`nom::ErrorKind`以及输入中报错的位置创建一个解析错误。如果`verbose-errors`属性没有激活，默认只打印错误代码。
- `fix_error!`: 将解析结果从`IResult`翻译为带有自定义类型的`IResult`

## 文本解析

- `escaped!`: 将字符串与转义字符相匹配
- `escaped_transform!`: 将字符串与转义字符相匹配，并返回一个转义字符被替换了的新字符串

## 二进制格式解析

- `length_data!`: 在第一个解析器中获取一个数字，然后获取输入中与其大小一样的子切片，并将其返回
- `length_bytes!`: length_data的别名
- `length_value!`: 在第一个解析器中获取一个数字，然后获取输入中与其大小一样的子切片，然后将第二个解析器应用于该子切片。如果第二个解析器返回`Incomplete`，`length_value!`将会返回一个错误

## 解析 Bit 流

- `bits!`: 将当前的输入类型(如：字节切片`&[u8]`)转换成一个 bit 流，可以在其上执行特定的 bit 解析器和其他的更通用的组合器
- `bytes!`: 将 bit 流输入转换成字节切片以支持基础解析器
- `tag_bits!`: 在 bit 流上进行整数模式匹配。必须指定要比较的输入位数
- `take_bits!`: 生成一个消耗指定位数的解析器

## 其他的组合器

- `apply!`: 模拟函数流动：`apply!(my_function, arg1, arg2, ...)`变成`my_function(input, arg1, arg2, ...)`
- `call!`: 将通用的表达式和函数封装成宏
- `method!`: 通过解析器组合生成一个方法
- `named!`: 通过解析器组合生成一个函数
- `named_args!`: 通过解析器组合生成一个带有参数的函数
- `named_attr!`: 通过解析器组合生成一个带有属性的函数
- `try_parse!`: 有点像`std::try!`，这个宏将会返回剩余的输入并当子解析器返回`Ok`时解析值，当子解析器返回`Error`或`Incomplete`时会提前返回。如果需要，它可以提供比`do_parse!`更复杂的功能
- `success`: 返回一个值但不消耗输入，总是解析成功

## 字符测试函数

将这些函数与类似`take_while!`的组合器一起使用：

- `is_alphabetic`: 测试字节是否是 ASCII 字母: `[A-Za-z]`
- `is_alphanumeric`: 测试字节是否是 ASCII 字母和数字: `[A-Za-z0-9]`
- `is_digit`: 测试字节是否是 ASCII 数字: `[0-9]`
- `is_hex_digit`: 测试字节是否是 ASCII 16进制数字: `[0-9A-Fa-f]`
- `is_oct_digit`: 测试字节是否是 ASCII 8进制数字: `[0-7]`
- `is_space`: 测试字节是否是 ASCII 空格或制表符: `[ \t]`
- `alpha`: 识别一个或多个小写和大写的字母: `[a-zA-Z]`
- `alphanumeric`: 识别一个或多个数字和字母: `[0-9a-zA-Z]`
- `anychar`: 
- `begin`: 
- `crlf`: 
- `digit`: 识别一个或多个数字: `[0-9]`
- `double`: 在字符串中识别浮点型数字并返回一个`f64`类型的值
- `eol`: 
- `float`: 在字符串中识别浮点型数字并返回一个`f32`类型的值
- `hex_digit`: 识别一个或多个16进制数字: `[0-9A-Fa-f]`
- `hex_u32`: 识别一个16进制编码数字
- `line_ending`: 识别行是否结束(包括`\n`和`\r\n`)
- `multispace`: 识别一个或多个空格，制表符，回车和换行符
- `newline`: 匹配换行符`\n`
- `non_empty`: 识别非空缓冲区
- `not_line_ending`: 
- `oct_digit`: 识别一个或多个8进制字符: `[0-7]`
- `rest`: 返回剩下的输入
- `shift`: 
- `sized_buffer`: 
- `space`: 识别一个或多个空格和制表符
- `tab`: 匹配制表符`\t`