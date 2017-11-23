---
layout: post
title: JavaScript代码规范
abstract: JavaScript代码规范
category: technical
permalink: javascript-code-style
author: 木逸辰
tags: [tech, javascript, code style]
---

### {{ page.title }}

整理一下`javascript`代码规范，基本上按照自己平时的习惯来，没洁癖写不了代码～

**1. 命名规范**

|        规则      |       示例       |
|------------------|-----------------|
| 变量、属性及函数的命名遵循驼峰命名法(camelCase) | `productChange` |
| 变量名可包含数字 | | `i18n` |
| 局部变量不能与更大作用域的变量重名 | N/A |
| 常量及枚举型使用全大写格式 | `CONSTANTS` `SETTING` |
| 构造函数首字母必须大写 | `FlightSearch` |
| 变量命名必须体现变量的意义 | `flights` `traveller` |
| 函数命名使用动词 | `search` `render` `travel` `transform` |
| 其他类型使用名词或形容词 | `traveller` `length` `valid` `transformer` |
| 数组等表示多个的变量使用复数形式(`data`、`list`除外) | `tickets` `passengers` |
| 变量命名在保证无歧义的条件下尽量精简，但只能使用约定俗成的缩写 | `len` `init` `param` `index` `idx` `utils` |
| 循环索引变量使用`i`、`j`、`k`、`idx`、`index`，不得使用其他名称| N/A |

*不得使用的变量形式*
- `_get`、`set_name` 等下划线开头的名称
- `string` `Object` `null` `undefined` `array` `number` 等关键字或保留字
- `a`、`b`、`a1` 等无具体含义的名称
- `obj`、`arr` 等非约定俗称的缩写
- `nameStr`、`ageNum` 变量名不得使用类型(或类型缩写)作为后缀

**注：一律不得使用错词别词，务必保证变量命名准确，发现错词别词必须立即修正**


**2. 排版规范**

- 代码缩进使用一个`tab`或者四个空格，`IDE`可设置自动将`tab`转换为四个空格
- 每条语句结尾必须带分号(`;`)
- 数组最后一个元素、对象最后一个属性之后不得带逗号(`,`)
- 二元及三元操作符两边必须有空格，一元操作符(`++` `--`)与变量之间不得有空格
- 函数的圆括号`)`与花括号`{`之间必须有空格
- 块代码(`if`、`while` etc.)如果多行书写，则必须有花括号(`{}`)，单行书写可不用
- 块级别嵌套不得超过三层，包括循环和条件判断
- 不得在块作用域和非顶层作用域范围内声明和定义函数，如：


                function change(valid){
                    if(valid){
                        function good(){

                        }
                    }
                    function bad(){

                    }
                }


**3. 编码规范**

- 无用的代码一律删除，不得保留或者注释
- 做相等比较时，一律使用严格等于(`===`)或严格不等于(`!==`)
- 禁止使用`label`标签
- 禁止使用`with`、`eval`
- 禁止使用`alert`、`prompt`、`confirm`
- 开发时可使用`console.log`调试，调试完毕必须删除此类代码
- 使用`parseInt`时必须指定第二个参数`radix`
- 浮点数必须完整，不得省略整数部分(`.12`)或小数部分(1.)
- 尽量不使用数字常数作为参数或表达式的一部分，应当为每个使用的常数定义一个常量变量
- 大量的字符串连接，尤其在循环体内的，尽量使用数组装载各部分子串，然后使用`join`方法组合

**4.注释规范**

- 短小的注释一律使用`//`，放在相应代码上方，`//`后面必须有一个空格如：`// 这是注释`
- 函数注释必须全面，如下（无返回值的函数也需要添加`@return`语句，返回为`void`）：

        /**
         * JSON数据的反序列化
         * 如果数据类型为string，则将其反序列化为JSON对象
         * @param {object|string} data 源数据
         * @return {object} 反序列化后的数据
         */
        json: function (data) {
            if (typeof data === "string")
                return JSON.parse(data);
            return data;
        }


**5.`nodejs`相关规范**

- 一律使用箭头函数
- 变量的声明和定义一律使用`let`和`const`，不得使用`var`
- 字符串组合尽量使用`template`语法(`this is a ${lang} template`)，不使用`+`拼接
- 合理使用`async`和`await`，不使用同步方法，回调函数的嵌套不能超过三层
- 路径组合一律使用`path.join()`方法，不得使用`+`拼接
