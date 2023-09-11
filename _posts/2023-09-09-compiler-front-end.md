---
layout: post
title: 编译器前端设计
category: [coding]
tag: [编译器]
author: programmer
publish: false
---

最近在计划实现 AutoScript 的编译器，但并不准备使用第三方库。前端部分可以通过一个*词法分析器*转化为词法单元流，随即再通过*语法分析器*转化为 **抽象语法树（Abstract Syntax Tree, AST）**。抽象语法树是可以直接求值的，因此从这里出发可以轻松地实现一个解释器。解释器可以用来方便地测试 AutoScript 的功能。

本篇文章会记录 AutoScript 编译器的前端，从源代码编译到 AST 的过程。

## 词法分析

词法分析会将一整个*编译单元*（即一个文件）中的代码转换成一系列词法单元，每个单元可能包含下面列出的信息之一：

* 一个字面量，比如整数字面量、字符串字面量等。
* 一个标识符，即变量的名字。
* 一个复合结构，比如表达式。

这里面，字面量拥有很好判断的格式，比如字符串总是以一个起始符（如 `"`）作为开始，随后接受一系列字符，并将第一个出现的结束符（如 `"`）作为结束的标志；整数字面量则是一系列数字组成的结构。这可以通过正则表达式轻松地匹配到。标识符也是同理的。

复合结构则是一类正则表达式无法分析的结构。比如一对圆括号 `()` 包含的结构是一个表达式，其中会包含零到多个词法单元。因此，词法分析器中需要一个能够支持递归的机制。

首先，让我们先规划一个可以处理字面量和标识符的词法分析器：

```cpp
class lexer_t {
public:
    bool consume(std::function<bool(char)>& pred) {	// 这里的 pred 没有以常引用方式传递，这是特地设置的
        // 复制一份 _begin，匹配失败时不会改变原来状态
        auto* _curr = _begin;
        while (pred(*_curr)) {
            ++curr;
        }
        auto match = _curr != _begin;
        _begin = _curr;
        return match;
    }
    
    bool is_completed() const noexcept {
        return _begin == _end;
    }
    
private:
    char const* _begin, _end;
};
```

这里我们使用 `_begin` 和 `_end` 存储下一个词法单元的开始位置和文件的结束位置。在 `consume` 函数中，我们使用参数 `pred` 来判断源代码是否能提取一个想要的词法单元。以字符串为例：

```cpp
auto const string_pred = [begin = false](char ch) mutable {
    // 初始需要一个双引号
    if (!begin) {
        begin = true;
        return ch == '"';
    }
    // 如果已经在字符串中，则遇到第一个双引号时终结这个字符串
    if (ch == '"') {
        return begin = false;
    }
    return true;
};
```

然后我们就可以像类似下面这样使用词法分析器：

```cpp
void lex(lexer_t lexer) {
    auto tokens = std::vector<token_t>();
    while (!lexer.is_completed()) {
        if (lexer.consume(int_pred)) {
			tokens.emplace_back(token_t);
        }
        if (lexer.consume(float_pred)) {
            lexer.consume(float_pred);
        }
        if (lexer.consume(string_pred)) {
			lexer.consume(string_pred);
        }
    }
}
```



