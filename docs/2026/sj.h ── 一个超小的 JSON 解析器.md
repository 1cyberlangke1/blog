最近在 [github](https://github.com/rxi/sj.h) 上看到了一个超小的 JSON 解析器, 用 C99 写的, 感觉很有意思

### 先看看头文件结构

只有一个头文件 `sj.h`, 我还是第一次知道 C语言也可以单头文件
结构如下
```C
#ifndef SJ_H
#define SJ_H
// 这里写声明
#endif // #ifndef SJ_H

#ifdef SJ_IMPL
// 这里写实现
#endif // #ifdef SJ_IMPL
```
用得时候就
```C
#define SJ_IMPL
#include "sj.h"
```
即可

### 下面来看看代码

看看声明
```C
// 解析器结构体: 存解析状态
typedef struct {
    char *data, *cur, *end; // 数据, 当前解析到的位置, 结尾 
    int depth; // 当前解析深度 (嵌套在第几层)
    char *error; // 错误信息
} sj_Reader;

// 值结构体: 存解析到的值的类型, 以及位置
typedef struct {
    int type; // 值的类型
    char *start, *end; // 数据开始和结束的位置
    int depth; // 值在嵌套第几层
} sj_Value;

// 枚举, 值的类型, 有错误, 结尾, 数组, 对象, 数字, 字符串, 布尔值, 空
enum { SJ_ERROR, SJ_END, SJ_ARRAY, SJ_OBJECT, SJ_NUMBER, SJ_STRING, SJ_BOOL, SJ_NULL };

// 初始化解析器
sj_Reader sj_reader(char *data, size_t len);
// 解析 JSON, 返回下一个值
sj_Value sj_read(sj_Reader *r);
// 数组迭代器, 迭代到最后返回 false
bool sj_iter_array(sj_Reader *r, sj_Value arr, sj_Value *val);
// 对象迭代器, 迭代到最后返回 false
bool sj_iter_object(sj_Reader *r, sj_Value obj, sj_Value *key, sj_Value *val);
// 查找当前解析位置在源文件的第几行几列
void sj_location(sj_Reader *r, int *line, int *col);
```

看看实现

```C
// 就是初始化, 使用了 C99 的特性
sj_Reader sj_reader(char *data, size_t len) {
    // 创建一个 sj_Reader 类型的匿名对象, 并对其初始化
    return (sj_Reader){ .data = data, .cur = data, .end = data + len };
}

// 判断字符是不是数字的一部分, 当前只是简单的判断, static 使其对外不可见, 这个是内部辅助函数
static bool sj__is_number_cont(char c) {
    return (c >= '0' && c <= '9')
        ||  c == 'e' || c == 'E' || c == '.' || c == '-' || c == '+';
}

// 内部辅助函数, 判断两个字符串是否相同
// cur: 当前位置, end: 结尾, expect: 另一个字符串的开头
static bool sj__is_string(char *cur, char *end, char *expect) {
    // 字符串结尾是 0, 所以 *expect 只要非零就会继续
    while (*expect) {
        // 当前字符串到结尾 (都不一样长肯定不一样了) 或者不一样就 false
        if (cur == end || *cur != *expect) {
            return false;
        }
        // 继续向后比较
        expect++, cur++;
    }
    // 不是 false 那就是 true 了
    return true;
}

// 最关键的部分, 解析器
sj_Value sj_read(sj_Reader *r) {
    // 返回值
    sj_Value res;
top:
    // 错误信息不是空指针, 说明有错误直接返回了
    if (r->error) { return (sj_Value){ .type = SJ_ERROR, .start = r->cur, .end = r->cur }; }
    // 到末尾了
    if (r->cur == r->end) { r->error = "unexpected eof"; goto top; }
    // 设置返回值的开头部分
    res.start = r->cur; 
    // 开始处理
    switch (*r->cur) {
    
    // 空格换行制表符, 这些直接跳过, 毕竟不是要的键值
    case ' ': case '\n': case '\r': case '\t':
    case ':': case ',':
        r->cur++;
        goto top; // 回顶继续往后遍历字符

    // 处理数字
    case '-': case '0': case '1': case '2': case '3': case '4':
    case '5': case '6': case '7': case '8': case '9':
        res.type = SJ_NUMBER; // 设置返回值是数字类型
        while (r->cur != r->end && sj__is_number_cont(*r->cur)) { r->cur++; } // 简单的遍历完数字, 不过显然处理不了 0.0.1 或者 -e6 之类的情况, 不过它简单且快
        break;

    // 处理字符串
    case '"':
        res.type = SJ_STRING; // 设置返回值是字符串
        res.start = ++r->cur; // 设置返回值的开头
        // 遍历字符串
        for (;;) {
            // 没找到另一个引号, 设置错误信息并回顶
            if ( r->cur == r->end) { r->error = "unclosed string"; goto top; }
            // 另一个引号, 到结尾了, 结束循环
            if (*r->cur ==    '"') { break; }
            // 如果是反斜杠, 说明是转义字符比如 '\"', 要跳过
            if (*r->cur ==   '\\') { r->cur++; }
            // 往后遍历
            if ( r->cur != r->end) { r->cur++; }
        }
        // 设置结尾并返回
        res.end = r->cur++;
        return res;

    // 对象或者数组判断
    // 左括号判断, 深度加1
    case '{': case '[':
        // 解析到底是对象还是数组
        res.type = (*r->cur == '{') ? SJ_OBJECT : SJ_ARRAY;
        res.depth = ++r->depth;
        r->cur++;
        break;
    // 右括号判断, 深度减1, 根据判断规则, 可以发现, 它会匹配 "{]", 这样的括号
    case '}': case ']':
        res.type = SJ_END; // 设置类型为结尾
        // 如果深度为 0, 匹配到右括号显然是错误的, 设置错误信息并回顶
        if (--r->depth < 0) {
            r->error = (*r->cur == '}') ? "stray '}'" : "stray ']'";
            goto top;
        }
        // 继续往后
        r->cur++;
        break;

    // 布尔值判断
    case 'n': case 't': case 'f':
        // 跳过开头判断是布尔类型还是空, 不过只检查开头
        res.type = (*r->cur == 'n') ? SJ_NULL : SJ_BOOL;
        // 跳过对应的部分
        if (sj__is_string(r->cur, r->end,  "null")) { r->cur += 4; break; }
        if (sj__is_string(r->cur, r->end,  "true")) { r->cur += 4; break; }
        if (sj__is_string(r->cur, r->end, "false")) { r->cur += 5; break; }
        // fallthrough

    // 没解析到也是错误
    default:
        r->error = "unknown token";
        goto top;
    }
    // 设置解析到的值的结尾
    res.end = r->cur;
    return res;
}

// 内部辅助函数, 跳过直到解析到想要的深度
static void sj__discard_until(sj_Reader *r, int depth) {
    sj_Value val;
    val.type = SJ_NULL;
    while (r->depth != depth && val.type != SJ_ERROR) {
        val = sj_read(r);
    }
}

// 数组迭代器
bool sj_iter_array(sj_Reader *r, sj_Value arr, sj_Value *val) {
    // 找到到数组部分
    sj__discard_until(r, arr.depth);
    // 获取数组的值
    *val = sj_read(r);
    // 判断结束
    if (val->type == SJ_ERROR || val->type == SJ_END) { return false; }
    // true 说明数组还有对象
    return true;
}

// 对象迭代器
bool sj_iter_object(sj_Reader *r, sj_Value obj, sj_Value *key, sj_Value *val) {
    // 找到到对象部分
    sj__discard_until(r, obj.depth);
    // 获取键
    *key = sj_read(r);
    if (key->type == SJ_ERROR || key->type == SJ_END) { return false; }
    // 获取值
    *val = sj_read(r);
    if (val->type == SJ_END)   { r->error = "unexpected object end"; return false; }
    if (val->type == SJ_ERROR) { return false; }

    return true;
}

// 寻找当前位置在文件几行几列, 用来获取具体错在哪里了
void sj_location(sj_Reader *r, int *line, int *col) {
    int ln = 1, cl = 1;
    // 就是遍历字符串
    for (char *p = r->data; p != r->cur; p++) {
        // 遇到换行就行 +1 列归零
        if (*p == '\n') { ln++; cl = 0; }
        cl++;
    }
    *line = ln;
    *col = cl;
}
```

### 总结
总之是一个 **设计很巧妙** 的 JSON 解析器, 虽然是弱检查, 解析所有像 JSON 的东西, 但是这个就 100 多行啊