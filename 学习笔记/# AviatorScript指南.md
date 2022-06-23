# AviatorScript指南
## 3 基本类型和语法
### 3.1 基本类型及运算
&emsp;&emsp;AviatorScript 支持常见的类型，如数字、布尔值、字符串等等，同时将大整数、BigDecimal、正则表达式也作为一种基本类型来支持

#### 3.1.1 整数和算术运算
&emsp;&emsp;整数类型提供long，没有int、short、byte，支持范围同java-9223372036854774808~9223372036854774807
&emsp;&emsp;包括+-*/%，运算规则同，运算优先级也相同
#### 3.1.2 大整数bigint
&emsp;&emsp;超过long范围的字面量，自动提升为bigint，对应java基类：java.math.BigInteger；任何以N结尾的数字，认为是bigint型
&emsp;&emsp;**long型计算结果超出范围，不糊自动提升为bigint，但是bigint和long一起参与运算的结果是bigint型**
#### 3.1.3 浮点型
&emsp;&emsp;仅支持double型，支持运算类型与整数相同，优先级也相同，运算结果为浮点型，整数和浮点型运算结果是浮点型
#### 3.1.4 高精度运算Decimal
&emsp;&emsp;以M结尾的字面量或者科学计数法，认为是decimal；decimal同样用+-*/做算术运算；除double外的所有数值类型和decimal运算结果都是decimal;double和decimal的运算结果是double---任何有double参与的运算，结果都是double
&emsp;&emsp;默认运算精度是 MathContext.DECIMAL128 ，你可以通过修改引擎配置项 Options.MATH_CONTEXT 来改变。
&emsp;&emsp;如果你觉的为浮点数添加 M 后缀比较麻烦，希望所有浮点数都解析为 decimal ，可以开启 Opitons.ALWAYS_PARSE_FLOATING_POINT_NUMBER_INTO_DECIMAL 选项
#### 3.1.5 数据类型转换
&emsp;&emsp;数据在运算时，遵循一定的规则：
 - 单一类型参与运算，结果仍为该类型
 - 多种类型参与的运算，按照：long->bigint->decimal->double自动提升
&emsp;&emsp;强转：long(X)将一个数字转换为long，可能会丢失精度，double(x) 讲一个数字转为double
#### 3.1.6 字符串：
&emsp;&emsp;以单引号或者双引号括起来的连续字符就是一个完整的字符串对象，长度可以使用string.length(a)来获取
&emsp;&emsp;可以使用+来拼接字符串，和java一样，任何类型和字符串相加，都视为字符串拼接
####  3.1.7  转义 
&emsp;&emsp;使用\来转义，也支持特殊字符 \r \n \t
#### 字符串插值
&emsp;&emsp;类似字符串format，但是支持更加灵活，使用#{}括起来的表达式，会在上下文中自动运算，并将结果插入最终字符串，示例
```
let name = "aviator";
let a = 1;
let b = 2;
let s = "hello, #{name}, #{a} + #{b} = #{a + b}";
```
#### 3.1.8 布尔值和逻辑运算
&emsp;&emsp;true和false 逻辑运算符包含{>、<、>=、<=、==、!=、&、|、!}

#### 1.1.9 逻辑运算和短路规则  TODO
&emsp;&emsp;使用&&或者||进行运算的时候，支持短路规则，既前 



###  3.4
#### 3.4.1 定义和赋值
&emsp;&emsp;变量定义和复制是不可分割的，定义一个变量的同时必须要给它赋值
&emsp;&emsp;可以使用type(x)函数来获取x的类型，返回值包含：

	- long 整数
	- double 浮点数
	- decimal 高精度数字类型
	- bigint 大整数
	- boolean 布尔类型
	- string 字符串类型
	- pattern 正则表达式
	- range 用于循环语句的 Range 类型
	- function 函数，参见第七章。
	- nil 特殊变量 nil，下文解释。
#### 3.4.2 动态类型
变量的类型会随着赋值而发生改变
#### 3.4.3 nil
nil表示一个变量还没有赋值，作用相当于null
```
a = nil;
if a == nil{
	printlbn(" a is " + type(a));
}
```
nil可以参与比较运算。任何类型都大于nil，除了nil本身；nil不能参与其他运算

#### 3.4.4 传入变量
通过execute(env)中的env来为表达式注入变量，传入的变量作用域默认是脚本内的全局变量，如果脚本中用到的变量没有传入，也没有定义，那么值将是nil
```
String expression = "a-(b-c) > 100";
Expression compiledExp = AviatorEvaluator.compile(expression);
// Execute with injected variables.
Boolean result =
      (Boolean) compiledExp.execute(compiledExp.newEnv("a", 100.3, "b", 45, "c", -199.100));
System.out.println(result);
```