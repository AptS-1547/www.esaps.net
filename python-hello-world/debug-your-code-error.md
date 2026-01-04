> 你的代码鼬鼬鼬鼬报错啦！
> 写代码当然少不了报错，关键是我们要学会怎么看
> --AptS:1549

## 错误……
错误，错误。
自从写代码开始就一直躲不过她
解释器给的红色字符，时时刻刻在刺痛我们的眼睛

[caption id="attachment_1733" align="aligncenter" width="921"]<img src="https://esaps.net/wp-content/uploads/2024/10/典型错误-1024x611.webp" alt="" width="921" height="577" class="size-large wp-image-1733" /> 这是一个小错误，但解释器直接报错[/caption]

但她其实并没有那么讨厌
她会直白的告诉你你的代码错在哪里，错的原因是什么
给你足够的信息去Debug
这就是我为什么这么喜欢错误信息，而不是你的源代码
**把解释器当成你的朋友，这是你并肩作战的伙伴**
[caption id="attachment_1011" align="aligncenter" width="300"]<img src="https://esaps.net/wp-content/uploads/2023/10/GF1FBX9J5_3AY_AM_YU__I-300x170.webp" alt="" width="450" height="255" class="size-medium wp-image-1011" /> 你的错误日志呢？[/caption]

## Python是怎么告诉你错在哪里的？
环境：Python 3.12.7 Windows 64-bit
```python
name = "Python"
print(f"it is {namee}")
-----------------------------
Traceback (most recent call last):
  File "c:UsersesapsDesktoperror.py", line 2, in <module>
    print(f"it is {namee}")
                   ^^^^^
NameError: name 'namee'; is not defined. Did you mean: 'name'?
-----------------------------
Python检测到错误后会告诉你代码的路径、行数，并指出是哪里错误，并给出错误类型
比如这里就是"NameError，变量'namee';，没有定义。你是不是在指'name'？"
解释器有时候可以给出修改建议给你，她真的，我哭死
```

接下来我将分类说明新手写代码比较常见的错误，以及如何使用错误信息进行Debug

-----------
### 语法错误 SyntaxError
当你输入的代码不符合Python的语法的时候会扔出这个错误
```bash
    if letter =="a" letter == "b":
                    ^^^^^^
SyntaxError: invalid syntax
-----------
语法错误，if判断过程中有多个条件需要用and、or等连接起来
-----------------------------
    if letter = "a" or letter == "b":
       ^^^^^^^^^^^^
SyntaxError: invalid syntax. Maybe you meant '==' or ':=' instead of '='?
-----------
语法错误，错误的使用了赋值符号，比较操作符号为两个等号（她甚至给了你解决方案，她真的，我哭死）
-----------------------------
    print("Hello Error Python!)
SyntaxError: EOL while scanning string literal
-----------
双引号没有闭合导致，把双引号补上
-----------------------------
#省略无关内容
if bian1==bian2==bian3：
#省略无关内容
-----------
SyntaxError: invalid character '：'(U+FF1A)
-----------
替换成英文字符即可
```

-----------
### ⚠️缩进同时使用空格和制表符 TabError
**请绝对不要同时使用空格和制表符！**
```python
# shape_name,num_sides已定义，shape_name函数返回一个字符串
if 3 <= num_sides <= 10:
    name = shape_name(num_sides)
	print(f"it is {name}")
-----------
Traceback (most recent call last):
    print(f"it is {name}")
TabError: inconsistent use of tabs and spaces in indentation
-----------
这段代码中，print函数前方使用的是制表符缩进，而其他都为4空格缩进。这个错误在3.6.3版本中很难查出来！
```

-----------
### 类型错误 TypeError
```python
a = 10
b = "5"
print(a + b)
-----------
TypeError: can only concatenate str (not "int") to str
-----------
解决方案：确保类型一致。
```

-----------
### 名称错误 NameError
```python
name = "Python"
print(f"it is {namee}")
-----------
NameError: name 'namee' is not defined. Did you mean: 'name'?
-----------
解决方案：确保变量在引用之前已被定义
```

-----------
### IndexError & KeyError 索引错误 & 键值错误
```python
lst = [1, 2, 3]
print(lst[3])
-----------
IndexError: list index out of range
-----------
解决方案：确保索引在列表范围内。

-----------------------------

my_dict = {"name": "Alice","age": 30}
print(my_dict["gender"])
-----------
KeyError: 'gender'
-----------
解决方案：确保键值在字典中。
```

-----------
### AttributeError 属性错误
```python
class MyClass:
    def __init__(self):
        self.value = 10

obj = MyClass()
print(obj.non_existent_attribute)
-----------
AttributeError: 'MyClass' object has no attribute 'non_existent_attribute'
-----------
解决方案：确保此类含有此属性。
```