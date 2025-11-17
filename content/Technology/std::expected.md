---
title: "C++23 std::expected：现代 C++ 的优雅错误处理"
date: 2025-11-17
tags: ["cpp", "cpp23", "error-handling", "programming"]
---

> 引言：错误处理一直是软件开发中的核心挑战。传统的 C++ 错误处理方式——异常和错误码——都有各自的局限性。异常可能带来性能开销和控制流的不透明，而错误码则容易被忽略且难以组合。C++23 引入的 `std::expected` 提供了一种类型安全、显式且零开销的错误处理方案，它借鉴了函数式编程语言（如 Rust 的 `Result<T, E>`）的设计理念。本文将深入探讨 `std::expected` 的设计原理、使用方法以及最佳实践。

## 什么是 std::expected?

`std::expected<T, E>` 是 C++23 标准库引入的一个模板类，定义在 `<expected>` 头文件中。它表示一个可能包含期望值（类型为 `T`）或错误值（类型为 `E`）的对象。这种设计使得函数的返回值能够显式地表达"成功或失败"的语义，而无需依赖异常或特殊的错误码。

### 核心特性

- **类型安全**：编译期就能确定返回值是成功还是失败
- **显式错误处理**：调用者必须显式检查结果，不会意外忽略错误
- **零开销抽象**：相比异常，没有额外的运行时开销
- **可组合性**：提供 `and_then`、`or_else`、`transform` 等函数式编程接口

## 传统错误处理的问题

### 1. 异常的问题

```cpp
// 使用异常的传统方式
int divide(int a, int b) {
    if (b == 0) {
        throw std::runtime_error("Division by zero");
    }
    return a / b;
}

int main() {
    try {
        int result = divide(10, 0);
        // 异常路径不明显
    } catch (const std::exception& e) {
        // 错误处理代码
    }
}
```

**问题：**
- 性能开销：异常机制需要额外的栈展开成本
- 控制流不明显：异常可能在任何地方抛出，难以追踪
- 不适合性能敏感场景：游戏、嵌入式系统等往往禁用异常

### 2. 错误码的问题

```cpp
// 使用错误码的传统方式
enum class ErrorCode {
    Success,
    DivisionByZero,
    Overflow
};

ErrorCode divide(int a, int b, int& result) {
    if (b == 0) {
        return ErrorCode::DivisionByZero;
    }
    result = a / b;
    return ErrorCode::Success;
}

int main() {
    int result;
    ErrorCode err = divide(10, 0, result);
    // 容易忘记检查错误码
    std::cout << result << std::endl; // result 未初始化！
}
```

**问题：**
- 容易被忽略：调用者可能忘记检查错误码
- 不方便组合：多个函数调用需要重复的错误检查
- 语义不清晰：返回值和输出参数混在一起

## 使用 std::expected 解决问题

### 基本用法

```cpp
#include <expected>
#include <string>

enum class MathError {
    DivisionByZero,
    Overflow
};

// 返回 expected：成功返回 int，失败返回 MathError
std::expected<int, MathError> divide(int a, int b) {
    if (b == 0) {
        return std::unexpected(MathError::DivisionByZero);
    }
    return a / b;
}

int main() {
    auto result = divide(10, 2);

    if (result.has_value()) {
        std::cout << "结果: " << result.value() << std::endl;
    } else {
        std::cout << "错误: " << static_cast<int>(result.error()) << std::endl;
    }
}
```

### 访问值的方式

```cpp
std::expected<int, MathError> result = divide(10, 2);

// 方式 1: 检查并访问
if (result) {  // 隐式转换为 bool
    std::cout << *result << std::endl;  // 解引用操作符
}

// 方式 2: value() 方法（失败时抛出异常）
try {
    std::cout << result.value() << std::endl;
} catch (const std::bad_expected_access<MathError>& e) {
    std::cout << "访问失败" << std::endl;
}

// 方式 3: value_or() 提供默认值
std::cout << result.value_or(0) << std::endl;

// 方式 4: 访问错误
if (!result) {
    MathError err = result.error();
}
```

### 函数式组合

`std::expected` 提供了强大的函数式接口，让错误处理更加优雅：

```cpp
// 链式调用，自动传播错误
std::expected<std::string, MathError> computeAndFormat(int a, int b) {
    return divide(a, b)
        .and_then([](int value) -> std::expected<int, MathError> {
            // 只在成功时执行，返回新的 expected
            return value * 2;
        })
        .transform([](int value) {
            // 转换成功值的类型
            return std::to_string(value);
        })
        .or_else([](MathError err) -> std::expected<std::string, MathError> {
            // 处理错误，可以恢复或转换错误
            return std::unexpected(err);
        });
}

int main() {
    auto result = computeAndFormat(10, 2);
    // 结果: "10" (10/2=5, 5*2=10, to_string)

    auto error_result = computeAndFormat(10, 0);
    // 自动传播 DivisionByZero 错误
}
```

## 实现原理剖析

`std::expected` 的实现基于一个 **tagged union**（带标签的联合体），它内部存储了一个布尔标志来区分当前是持有值还是错误。

### 简化的实现模型

```cpp
template<typename T, typename E>
class expected {
private:
    union {
        T value_;
        E error_;
    };
    bool has_value_;

public:
    // 构造成功值
    constexpr expected(const T& value)
        : value_(value), has_value_(true) {}

    // 构造错误值
    constexpr expected(const std::unexpected<E>& err)
        : error_(err.error()), has_value_(false) {}

    // 检查是否有值
    constexpr bool has_value() const noexcept {
        return has_value_;
    }

    // 获取值（假设已检查）
    constexpr const T& value() const& {
        if (!has_value_) {
            throw std::bad_expected_access(error_);
        }
        return value_;
    }

    // 获取错误（假设已检查）
    constexpr const E& error() const& {
        return error_;
    }

    // 析构函数需要根据标志析构正确的成员
    ~expected() {
        if (has_value_) {
            value_.~T();
        } else {
            error_.~E();
        }
    }
};
```

### 关键设计点

1. **零开销抽象**：`expected` 的大小通常是 `sizeof(T)` 和 `sizeof(E)` 中较大者加上一个布尔标志（考虑对齐）。没有堆分配，没有虚函数，性能接近原始类型。

2. **std::unexpected**：用于明确表示错误值的包装类型，避免构造函数歧义：
```cpp
// 没有 unexpected，这会产生歧义
std::expected<int, int> result = 42;  // 是成功值还是错误值？

// 使用 unexpected 明确表示错误
std::expected<int, int> result = std::unexpected(42);  // 明确是错误
```

3. **类型安全的析构**：由于使用 union，析构函数必须根据 `has_value_` 标志选择性地调用正确的析构函数。

## 实际应用场景

### 1. 文件操作

```cpp
#include <expected>
#include <fstream>
#include <string>

enum class FileError {
    NotFound,
    PermissionDenied,
    ReadError
};

std::expected<std::string, FileError> readFile(const std::string& path) {
    std::ifstream file(path);

    if (!file.is_open()) {
        return std::unexpected(FileError::NotFound);
    }

    std::string content;
    std::string line;
    while (std::getline(file, line)) {
        content += line + "\n";
    }

    if (file.bad()) {
        return std::unexpected(FileError::ReadError);
    }

    return content;
}

// 使用示例
void processFile(const std::string& path) {
    auto content = readFile(path)
        .and_then([](const std::string& data) -> std::expected<size_t, FileError> {
            // 处理文件内容
            return data.size();
        })
        .transform([](size_t size) {
            return size * 2;
        });

    if (content) {
        std::cout << "处理后的大小: " << *content << std::endl;
    } else {
        std::cout << "文件错误" << std::endl;
    }
}
```

### 2. 网络请求

```cpp
enum class NetworkError {
    ConnectionFailed,
    Timeout,
    InvalidResponse
};

struct HttpResponse {
    int status_code;
    std::string body;
};

std::expected<HttpResponse, NetworkError> httpGet(const std::string& url) {
    // 模拟网络请求
    if (url.empty()) {
        return std::unexpected(NetworkError::InvalidResponse);
    }

    return HttpResponse{200, "Response body"};
}

std::expected<std::string, NetworkError> fetchAndParse(const std::string& url) {
    return httpGet(url)
        .and_then([](const HttpResponse& resp) -> std::expected<std::string, NetworkError> {
            if (resp.status_code != 200) {
                return std::unexpected(NetworkError::InvalidResponse);
            }
            return resp.body;
        })
        .transform([](const std::string& body) {
            // 解析响应
            return "Parsed: " + body;
        });
}
```

### 3. 配置解析

```cpp
#include <expected>
#include <string>
#include <map>
#include <charconv>  // C++17 零开销解析

enum class ConfigError {
    ParseError,
    MissingKey,
    InvalidType
};

class Config {
    std::map<std::string, std::string> data_;

public:
    std::expected<int, ConfigError> getInt(const std::string& key) const {
        auto it = data_.find(key);
        if (it == data_.end()) {
            return std::unexpected(ConfigError::MissingKey);
        }

        // 使用 std::from_chars 而非 std::stoi
        int result;
        const auto& str = it->second;
        auto [ptr, ec] = std::from_chars(str.data(), str.data() + str.size(), result);

        if (ec != std::errc{} || ptr != str.data() + str.size()) {
            return std::unexpected(ConfigError::InvalidType);
        }

        return result;
    }

    std::expected<std::string, ConfigError> getString(const std::string& key) const {
        auto it = data_.find(key);
        if (it == data_.end()) {
            return std::unexpected(ConfigError::MissingKey);
        }
        return it->second;
    }
};

// 链式获取和验证配置
std::expected<int, ConfigError> getValidatedPort(const Config& config) {
    return config.getInt("port")
        .and_then([](int port) -> std::expected<int, ConfigError> {
            if (port < 1024 || port > 65535) {
                return std::unexpected(ConfigError::InvalidType);
            }
            return port;
        });
}
```

### 4. 数据库操作

```cpp
enum class DbError {
    ConnectionFailed,
    QueryFailed,
    NoResults
};

struct User {
    int id;
    std::string name;
};

class Database {
public:
    std::expected<User, DbError> findUserById(int id) {
        // 模拟数据库查询
        if (id <= 0) {
            return std::unexpected(DbError::QueryFailed);
        }

        if (id > 1000) {
            return std::unexpected(DbError::NoResults);
        }

        return User{id, "User" + std::to_string(id)};
    }

    std::expected<void, DbError> updateUser(const User& user) {
        // std::expected<void, E> 用于没有返回值但可能失败的操作
        if (user.name.empty()) {
            return std::unexpected(DbError::QueryFailed);
        }
        return {};  // 成功，无返回值
    }
};

// 事务式操作
std::expected<User, DbError> updateAndReturn(Database& db, int userId, const std::string& newName) {
    return db.findUserById(userId)
        .and_then([&](User user) -> std::expected<User, DbError> {
            user.name = newName;
            return db.updateUser(user)
                .and_then([user]() -> std::expected<User, DbError> {
                    return user;
                });
        });
}
```

## 最佳实践和注意事项

### Do's:

1. **使用有意义的错误类型**
```cpp
// 好的做法：使用枚举或结构体表示具体错误
enum class ValidationError {
    EmptyInput,
    TooLong,
    InvalidCharacters
};

std::expected<std::string, ValidationError> validateInput(const std::string& input);

// 避免：使用泛型类型
std::expected<std::string, std::string> validateInput(const std::string& input);  // 不推荐
```

2. **优先使用函数式组合**
```cpp
// 好的做法：使用 and_then、transform 链式调用
auto result = parseInput(input)
    .and_then(validate)
    .transform(process)
    .value_or(defaultValue);

// 避免：嵌套的 if-else
auto parsed = parseInput(input);
if (parsed) {
    auto validated = validate(*parsed);
    if (validated) {
        auto processed = process(*validated);
        // ...
    }
}
```

3. **使用 `std::expected<void, E>` 表示可能失败的无返回值操作**
```cpp
std::expected<void, Error> initialize() {
    if (!setup()) {
        return std::unexpected(Error::InitFailed);
    }
    return {};  // 成功
}
```

4. **为错误类型提供有用的上下文**
```cpp
struct FileError {
    enum class Kind {
        NotFound,
        PermissionDenied
    };
    Kind kind;
    std::string path;
    std::string message;
};

std::expected<std::string, FileError> readFile(const std::string& path);
```

### Don'ts:

1. **不要使用异常作为错误类型**
```cpp
// 错误：混合异常和 expected
std::expected<int, std::exception> divide(int a, int b);  // 不推荐

// 应该使用具体的错误枚举
enum class MathError { DivisionByZero };
std::expected<int, MathError> divide(int a, int b);
```

2. **不要忽略错误**
```cpp
// 错误：直接访问可能失败的结果
auto result = divide(10, 0);
std::cout << *result << std::endl;  // 未检查，可能未定义行为

// 正确：总是检查
if (result) {
    std::cout << *result << std::endl;
}
```

3. **不要过度使用 `value()`**
```cpp
// 不推荐：value() 失败时抛异常，违背了 expected 的初衷
try {
    auto val = divide(10, 0).value();
} catch (...) {
    // 这和使用异常没什么区别
}

// 推荐：使用 if 检查或 value_or
if (auto result = divide(10, 0)) {
    auto val = *result;
}
```

4. **不要在性能不敏感的地方过度使用**
```cpp
// 对于简单的内部函数，不需要使用 expected
std::expected<int, Error> addOne(int x) {  // 过度设计
    return x + 1;
}

// 简单函数直接返回即可
int addOne(int x) {
    return x + 1;
}
```

### 与异常的对比

| 场景 | 使用 expected | 使用异常 |
|------|--------------|----------|
| 性能敏感代码 | ✅ | ❌ |
| 错误是预期的一部分 | ✅ | ❌ |
| 需要禁用异常的环境 | ✅ | ❌ |
| 深层调用栈的错误传播 | ❌ | ✅ |
| 不可恢复的错误 | ❌ | ✅ |
| 构造函数失败 | ❌ | ✅ |

## 与其他语言的对比

`std::expected` 的设计借鉴了多种语言的经验：

### Rust 的 Result<T, E>
```rust
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err("Division by zero".to_string())
    } else {
        Ok(a / b)
    }
}

// 使用 ? 操作符自动传播错误
fn compute() -> Result<i32, String> {
    let result = divide(10, 2)?;
    Ok(result * 2)
}
```

C++ 的 `expected` 提供了类似的功能，但使用 `and_then` 代替 `?` 操作符。

### Haskell 的 Either
```haskell
divide :: Int -> Int -> Either String Int
divide _ 0 = Left "Division by zero"
divide a b = Right (a `div` b)
```

### Swift 的 Result
```swift
enum MathError: Error {
    case divisionByZero
}

func divide(_ a: Int, _ b: Int) -> Result<Int, MathError> {
    guard b != 0 else {
        return .failure(.divisionByZero)
    }
    return .success(a / b)
}
```

## 性能考量

`std::expected` 的一个主要优势是零开销抽象：

```cpp
// 编译器通常会将 expected 优化为与手写错误码相同的性能
std::expected<int, Error> divide(int a, int b) {
    if (b == 0) return std::unexpected(Error::DivisionByZero);
    return a / b;
}

// 等价于传统的错误码方式（性能相同）
int divide_old(int a, int b, int& result) {
    if (b == 0) return -1;
    result = a / b;
    return 0;
}
```

**基准测试对比**（相对性能）：
- `std::expected`: ~1x （基准）
- 错误码: ~1x （相同）
- 异常（成功路径）: ~1x
- 异常（失败路径）: ~100-1000x （取决于栈深度）

## 结论

`std::expected` 为 C++ 带来了一种现代化、类型安全且零开销的错误处理机制。它结合了异常的表达力和错误码的性能，特别适合以下场景：

1. **性能敏感的应用**：游戏引擎、高频交易系统、嵌入式系统
2. **错误是常见情况**：文件 I/O、网络请求、用户输入验证
3. **需要显式错误处理**：API 设计、库开发
4. **函数式编程风格**：需要链式调用和错误传播

通过合理使用 `std::expected`，我们可以编写出更安全、更清晰且性能更优的 C++ 代码。记住关键原则：使用有意义的错误类型，善用函数式组合，并在适当的场景下选择 `expected` 而非异常。

## 参考资料

- [C++23 Standard - std::expected](https://en.cppreference.com/w/cpp/utility/expected)
- [P0323R12: std::expected proposal](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2022/p0323r12.html)
- [Rust Book - Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
