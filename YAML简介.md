# YAML简介

YAML是”YAML Ain't markup language"（YAML不是一种标记语言）的缩写，是一种对人类设计友好（方便读写）的数据序列化语言，可以很好地与其它编程语言协同完成日常任务。

它是JSON的一个严格超集，在语法上增加了类似Python的换行和缩进。不过，与Python不同，YAML不允许使用Tab缩进。

## 基本规则

YAML有一些基本的规则，用来避免与各种编程语言和编辑器相关的歧义问题，这些基本的规则使得无论哪个应用程序或软件库都能一致地解析YAML。

- 文件名以`.yaml`结尾
- 大小写敏感
- 不允许使用Tab。由于Tab的支持性不够普通，因此使用空格。

## 基本数据类型

YAML的基本数据类型与JSON一致，字符串、数字等标量类型，List、Map等容器类型。

YAML擅长处理**映射表**(哈希表 / 字典)、**序列**(数组 / 列表)和**标量**(字符串 / 数字)。

### 标量

最基础的数据类型，包括布尔值、数字、字符串。

```yaml
integer: 25
string: "25"
float: 25.0
boolean: Yes
```

### 序列

列表或者数组。每个item一行，以`-`开头。

```
- Cat
- Dog
- Goldfish
```

同一层级的item是一个序列，可以用缩进来表示多层的序列。

```yaml
-
  - Cat
  - Dog
  - Goldfish
-
  - Python
  - Lion
  - Tiger
```

可以是多层的：

```yaml
-
 -
  - Cat
  - Dog
  - Goldfish
```

也可以这么写：

```yaml
--- Cat
  - Dog
  - Goldfish
```

### 映射表

键值对，如：

```yaml
animal: pets
```

如果值是一个序列：

```yaml
pets:
  - Cat
  - Dog
  - Goldfish
```

## 5分钟教程

```yaml
--- # 文档开头

# YAML注释类似这样

################
#    标量类型   #
################

# 根对象是一个map，相当于其它语言中的dictionary、hash或者object。
key: value
another_key: Another value goes here.
a_number_value: 100
scientific_notation: 1e+12

# 数字1会被解析为数值类型，而不是布尔类型。如果希望被解析为布尔类型，请使用`true`
boolean: true
null_value: null
key with spaces: value

# 字符串不需要使用引号，不过可以使用。
however: 'A string, enclosed in quotes.'
'Keys can be quoted too.': "Useful if you want to put a ':' in your key."
single quotes: 'have ''one'' escape pattern'
double quotes: "have many: \", \0, \t, \u263A, \x0d\x0a == \r\n, and more."
# UTF-8/16/32的字符需要编码
Superscript two: \u00B2

# 多行字符串可以写作"literal block"（用'|'）或者"folded block"（用'>'）
literal_block: |
    整块文本都是'literal_block'的值，换行会被保留。

    只要缩进的文本都包含在block中，首缩进会被去除。
    
        多余的缩进会被保留 - 此行文字会缩进4个空格。
        
folded_style: >
    整块文本都是`folded_style`的值，不过所有的换行会被替换为一个。
    
    上面的空行会被转换为一个换行符。
    
        多余的缩进依然会保留其换行 -
        此段文字会展示为两行。
        
####################
#      容器类型     #
####################

# 嵌套使用缩进。推荐使用2个空格的缩进（但不是必须的）。
a_nested_map:
  key: value
  another_key: Another Value
  another_nested_map:
    hello: hello
    
# Map的key可以是非字符串
0.25: a float key

# Key可以是复杂对象，像多行对象一样，我们使用问号后跟一个空格来表示复杂Key的开始。
? |
  This is a key
  that has multiple lines
: and this is its value

# YAML还允许使用复杂Key语法在序列之间进行映射
# 某些语言的解析器可能会有警告
? - Manchester United
  - Real Madrid
: [2001-01-01, 2002-02-02]

# 序列（List、Array等）这样表示
# (注意，'-'算作缩进)
a_sequence:
  - Item 1
  - Item 2
  - 0.5  # 序列可以包含不同的类型
  - Item 4
  - key: value
    another_key: another_value
  -
    - This is a sequence
    - inside another sequence
  - - - Nested sequence indicators
      - can be collapsed
      
# YAML是JSON的严格超集，因此可以使用JSON风格的map和序列
json_map: {"key": "value"}
json_seq: [3, 2, 1, "takeoff"]
and quotes are optional: {key: [3, 2, 1, takeoff]}

#######################
#    YAML的额外特性    #
#######################

# YAML有一个方便特性“anchor"，可以方便地在文档中重复内容。以下两个key对应的值相同：
anchored_content: &anchor_name This string will appear as the value of two keys.
other_anchor: *anchor_name

# Anchor可以用来重复/继承属性
base: &base
  name: Everyone has same name

# 正则表达式 << 称作Merge Key Language-Independent Type，
# 表示将一个或多个map的所有key插入到当前map

foo: &foo
  <<: *base
  age: 10

bar: &bar
  <<: *base
  age: 20

# foo和bar都会包含name: Everyone has same name

# YAML还支持tag，用来显式地声明类型。
explicit_string: !!str 0.5 # 指明字符串类型
# 某些解析器实现了语言相关的tag，比如Python的复杂数值类型
python_complex_number: !!python/complex 1+2j

# 语言相关的tag可以与复杂key一起使用
? !!python/tuple [5, 7]
: Fifty Seven
# 在Python中值为{(5, 7): 'Fifty Seven'}

####################
#   YAML的额外类型  #
####################

# YAML不仅可以理解字符串和数字这样的基本类型，
# 也可以理解ISO格式的日期和时间字面量。
datetime: 2001-12-15T02:59:43.1Z
datetime_with_spaces: 2001-12-14 21:59:43.10 -5
date: 2002-12-14

# !!binary这个tag表示一个base64编码的二进制对象
gif_file: !!binary |
  R0lGODlhDAAMAIQAAP//9/X17unp5WZmZgAAAOfn515eXvPz7Y6OjuDg4J+fn5
  OTk6enp56enmlpaWNjY6Ojo4SEhP/++f/++f/++f/++f/++f/++f/++f/++f/+
  +f/++f/++f/++f/++f/++SH+Dk1hZGUgd2l0aCBHSU1QACwAAAAADAAMAAAFLC
  AgjoEwnuNAFOhpEMTRiggcz4BNJHrv/zCFcLiwMWYNG84BwwEeECcgggoBADs=

# YAML也支持集合类型：
set:
  ? item1
  ? item2
  ? item3
or: {item1, item2, item3}

# 集合其实是值为null的map；上面的写法等同于：
set2:
  item1: null
  item2: null
  item3: null

...  # 文档结尾
```

