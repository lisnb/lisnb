---
title: 一种在C++中进行JSON字段校验的方法[3]
date: 2017-04-19 19:33:53
categories: C++
tags: 
- json
- validation
---

上一篇中说明了实现的思路，这篇中就实现一个用来在 C++ 中进行 JSON 字段校验的校验器。

<!--more-->

我们希望最终应用于校验器的配置文件包含两个部分：
1. 校验逻辑
2. 校验所用的参数

对于第1篇给出的 json 示例：
作为例子，假设服务器返回的结果是如下的结构：

```json
{
  "charlie": -1,
  "delta": 4.2,
  "foxtrot": false,
  "golf": {
    "hotel": "india",
    "july": ["kilo", "lima", "mike"],
    "november": {
      "oscar": {
        "papa": null
      }
    }
  }
}
```
- ```charlie``` 大于1且小于10
- ```delta``` 是浮点型或者等于0.0
- ```"mike"``` 在 ```july``` 中

我们对于上面的验证条件给出的配置文件的内容是

``` json
{
    "charlie": {
        "$gt": 1,
        "$lt": 10
    },
    "$or": [
        {
            "delta": {
                "$is_double": true
            }
        },{
            "delta": {
                "$eq": 0.0
            }
        }
    ],
    "golf.july": {
        "$has": "mike"
    }
}
```

## 获取嵌套字段的值
将提供的字段名通过 "." 分开，然后一层一层的获取 

```C++
// 这里假定所有的值都能找到
bool GetFieldValue(
    const std::string &field_name,
    const Json::Value &data) {
  const Json::Value *value(&data);
  std::vector<std::string> fields = some_split_function(field_name);
  for (auto field_cit = fields.cbegin(); filed_cit != fields.cend(); ++field_cit) {
    value = &(*value)[*field_cit];
  }
  return *value;
}
```

## 基本的运算符
实现的基本运算符需要放在一个 map 里，所以每个运算符都是一个类，继承同一个基类
``` C++
// 所有基本运算符的基类，重载 () 运算符
class BasicOperator {
 public:
  virtual bool operator()(
      const Json::Value &field, 
      const Json::Value &arg) const = 0;
};

// 大于函数
class gt : public BasicOperator {
 public:
  overide bool operator()(
      const Json::Value &field,
      const Json::Value &arg) const {
    // 直接调用 Json::Value 重载的运算符
    return field > arg
  }
};

//小于函数(lt)、相等(eq)和大于函数类似，略过

// double 类型判断
class is_double : public BasicOperator {
 public:
  overide bool opeartor()(
      const Json::Value &field,
      const Json::Value &arg) {
    // arg 虽然在配置文件中配置为 true， 但在这里也是 Json::Value 类型
    // 需要使用 asBool 转换成 bool 型才能进行判断
    return field.isDouble() == arg.asBool();
  }
}

// has
class has : public BasicOperator {
 public:
  overide bool opeartor()(
      const Json::Value &field,
      const Json::Value &arg) { 
    for (auto cit = field.cbegin(); cit != field.cend(); ++cit) {
      if (arg == *cit) {
        return true;
      }
    }
    return false;
  }
}
```

## 逻辑运算符
逻辑运算符共同继承另一个基类：

```C++
// 所有逻辑运算符的基类，重载 () 运算符
class LogicalOperator {
 public:
  virtual bool operator()(
      const Json::Value &validators, 
      const Json::Value &data) const = 0;
};

// 

```


