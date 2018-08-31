---
title: 【译】PEP 484 -- 类型提示（Type Hints）
date: 2018-05-24 15:04:53
tags:
    - PEP
    - Python
    - Type Hints
description: PEP 484 -- Type Hints(类型提示) 中文版（未完成，暂时使用机翻）
---
# PEP 484 - 类型提示

PEP       | 484
:-------  | :-------------
标题      | 键入提示（Type Hints）
作者      | <a href="mailto:guido@python.org">Guido van Rossum</a>, <a href="mailto:jukka.lehtosalo@iki.fi">Jukka Lehtosalo</a>, <a href="mailto:lukasz@python.org">Łukasz Langa</a>
BDFL-代表 | Mark Shannon
讨论组    | <a href="mailto:python-dev@python.org">Python-Dev</a>
状态      | 已接受
类型      | 标准跟踪
创建于    | 2014年9月29日
Python版本 | 3.5
修改历史   | 2015年1月16日，2015年3月20日，2015年4月17日，2015年5月20日，2015年5月22日
正式决定   | [https://mail.python.org/pipermail/python-dev/2015-May/140104.html](https://mail.python.org/pipermail/python-dev/2015-May/140104.html)

## 摘要

[PEP 3107](https://www.python.org/dev/peps/pep-3107)引入了函数注释的语法，但是有意留下了语义未曾定义。现在已经有足够多的第三方使用静态类型分析，社区将从标准库中的标准词汇和基准工具中受益。

该PEP引入了一个临时模块来提供这些标准定义和工具，以及一些针对注释不可用的情况的约定。

请注意，此PEP显然不会妨碍注释的其他用途，也不会要求（或禁止）任何特定的注释处理，即使它们符合此规范。它只是促进更好的协调，正如[PEP 333](https://www.python.org/dev/peps/pep-0333)为Web框架所做的那样。

例如，下面是一个简单的函数，其参数和返回类型在注释中声明：

```python
def greeting(name: str) -> str:
    return 'Hello ' + name
```

尽管这些注释在运行时通过常用的`__annotations__`属性可以获取到，*但在运行时不会执行类型检查*。相反，该提议假定存在单独的离线型检查器，用户可以自愿运行其源代码。本质上，这种类型检查器是一个非常强大的linter。（当然，对于单个用户来说，可以在运行时使用类似的检查器来执行Design by Contract或JIT优化，但这些工具尚未成熟。）

该提案受到[mypy](https://www.python.org/dev/peps/pep-0484/#mypy)的强烈启发。例如，`sequence of integers`类型可以写为`Sequence [int]`。方括号表示不需要将新语法添加到该语言中。这里的示例使用从纯Python模块`typing`导入的自定义类型`Sequence`。通过在元类中实现`__getitem __()`，`Sequence [int]`表示法可以在运行时工作（但它的意义主要是离线类型检查器）。

类型系统支持联合、泛型类型和命名`any`的与所有类型一致的(即赋值)的特殊类型。后一种特征是从`gradual typing`的概念中提取出来的。在[PEP 483](https://www.python.org/dev/peps/pep-0483)中解释了`gradual typing`和完整类型系统。

[PEP 482](https://www.python.org/dev/peps/pep-0482)中描述了我们所借鉴的其他方法或我们可以比较和对比的方法。

## 理由和目标

[PEP 3107](https://www.python.org/dev/peps/pep-3107)增加了对函数定义部分的任意注释的支持。虽然没有给注释分配任何意义，但是总是有一个隐含的目标，就是将它们用于类型提示[gvr-artima](https://www.python.org/dev/peps/pep-0484/#gvr-artima)，它被列为所述PEP中的第一个可能的用例。

此PEP旨在为类型注释提供标准语法，打开Python代码以更轻松地进行静态分析和重构，潜在的运行时类型检查以及（可能在某些情况下）使用类型信息的代码生成。

在这些目标中，静态分析是最重要的。这包括对离线类型检查程序（如mypy）的支持，以及提供IDE可用于代码完成和重构的标准符号。

### 非目标

尽管提议的`typing`模块将包含用于运行时类型检查的一些构建块，特别是`get_type_hints()`函数，但第三方包必须被开发来实现特定的运行时类型检查功能，例如使用装饰器或元类。使用类型提示进行性能优化是读者的一项练习。

还应该强调的是，**Python仍然是一种动态类型的语言，即使按惯例，作者也不希望强制类型提示**。

## 注释的含义

任何没有注释的函数应该被视为具有可能的最普通类型，或被任何类型检查器忽略。带有`@no_type_check`装饰器的函数应该被视为没有注释。

建议但不要求checked函数对所有参数和返回类型都有注释。对于checked函数，参数和返回类型的默认注释是Any。例外是实例和类方法的第一个参数。如果未注释，则假定它具有实例方法的包含类的类型，以及与类方法的包含类对象相对应的类型对​​象类型。例如，在类A中，实例方法的第一个参数具有隐式类型A.在类方法中，第一个参数的精确类型不能用可用类型表示法表示。

（请注意，`__init__`的返回类型应该用 `-> None`注释，原因很简单，如果`__init__`假定返回注释为 `-> None`，那么这意味着一个无参数的，未注释的`__init__`方法应该仍然需要进行类型检查吗？我们只是简单地说`__init__`应该有一个返回注释;因此默认行为与其他方法相同，而不是留下这种模糊不清或引入异常的异常。）

期望类型检查器检查被检查函数的主体是否与给定的注释一致。注释也可用于检查在其他检查功能中出现的呼叫的正确性。

预计类型检查人员将尝试根据需要推断尽可能多的信息。最低要求是处理内置装饰器`@property`，`@staticmethod`和`@classmethod`。

## 类型定义语法

该语法利用[PEP 3107](https://www.python.org/dev/peps/pep-3107)样式注释以及下面部分中描述的许多扩展。在其基本形式中，类型提示用于填充具有以下类的函数注释槽：

```python
def greeting(name: str) -> str:
    return 'Hello ' + name
```

这表明名称参数的预期类型是`str`。类似地，预期的回报类型是`str`。

表达式的类型是特定参数类型的子类型，也被该参数接受。

### 可接受的类型提示（Acceptable type hints）

类型提示可以是内置类（包括在标准库或第三方扩展模块中定义的类），抽象基类，`typing`模块中可用的类型和用户定义的类（包括在标准库或第三方模块中定义的那些）。

尽管注释通常是用于提示类型的最佳格式，但有些时候更适合用特殊注释或独立分布的存根文件来表示它们。 （请参阅下面的示例。）

注释必须是有效的表达式，在函数定义时不会引发异常而进行评估（但请参阅下面的前向引用）。

注释应该保持简单，否则静态分析工具可能无法解释这些值。例如，动态计算类型不太可能被理解。 （这是一个有意含糊的要求，具体包含和排除可能会被添加到该PEP的未来版本中，如讨论所保证的。）

除上述以外，还可以使用下面定义的特殊结构：`None`，`Any`，`Union`，`Tuple`，`Callable`，从`typing`（例如`Sequence`和`Dict`）导出的具体类的所有ABCs和替身，类型变量和类型别名。

`typing`模块中提供了所有新引入的用于支持以下各节（如`Any`和`Union`）中描述的功能的名称。

### 使用None

在类型提示中使用时，表达式`None`被认为与`type(None)`等效。

### 类型别名（Type aliases）

类型别名由简单变量赋值来定义：

```python
Url = str

def retry(url: Url, retry_count: int) -> None: ...
```

请注意，我们建议大写别名，因为它们代表用户定义的类型，这些类型（如用户定义的类）通常以这种方式拼写。

类型别名可能与注释中的类型提示一样复杂 - 在类型别名中可接受的任何类型提示都是可接受的：

```python
from typing import TypeVar, Iterable, Tuple

T = TypeVar('T', int, float, complex)
Vector = Iterable[Tuple[T, T]]

def inproduct(v: Vector[T]) -> T:
    return sum(x*y for x, y in v)
def dilate(v: Vector[T], scale: T) -> Vector[T]:
    return ((x * scale, y * scale) for x, y in v)
vec = []  # type: Vector[float]
```

这相当于：

```python
T = TypeVar('T', int, float, complex)

def inproduct(v: Iterable[Tuple[T, T]]) -> T:
    return sum(x*y for x, y in v)
def dilate(v: Iterable[Tuple[T, T]], scale: T) -> Iterable[Tuple[T, T]]:
    return ((x * scale, y * scale) for x, y in v)
vec = []  # type: Iterable[Tuple[float, float]]
```

### 回调（Callable）

期望特定签名的回调函数的框架可以使用`Callable[[Arg1Type, Arg2Type], ReturnType]`进行类型提示。例子：

```python
from typing import Callable

def feeder(get_next_item: Callable[[], str]) -> None:
    # Body

def async_query(on_success: Callable[[int], None],
                on_error: Callable[[int, Exception], None]) -> None:
    # Body
```

可以通过用文字省略号（三个点）代替参数列表来声明可调用的返回类型，而无需指定调用签名：

```python
def partial(func: Callable[..., str], *args) -> Callable[..., str]:
    # Body
```

请注意，省略号周围没有方括号。回调的参数在这种情况下是完全不受限制的（关键字参数是可以接受的）。

由于使用带关键字参数的回调函数不被视为常见用例，因此目前不支持使用`Callable`指定关键字参数。同样，不支持使用特定类型的可变数量的参数来指定回调签名。

由于`typing.Callable`执行双重职责，作为`collections.abc.Callable`的替代品，`isinstance(x, typing.Callable)`通过推迟到`isinstance(x, collections.abc.Callable)`实现。但`isinstance(x, typing.Callable[...])`不受支持。

### 泛型（Generics）

由于无法以通用方式静态推断有关保存在容器中的对象的类型信息，因此抽象基类已扩展为支持订阅以表示容器元素的预期类型。例：

```python
from typing import Mapping, Set

def notify_by_email(employees: Set[Employee], overrides: Mapping[str, str]) -> None: ...
```

泛型可以通过使用`typing`中名为`TypeVar`的新工厂进行参数化。例：

```python
from typing import Sequence, TypeVar

T = TypeVar('T')      # Declare type variable

def first(l: Sequence[T]) -> T:   # Generic function
    return l[0]
```

在这种情况下，合同是返回的值与集合中保存的元素一致。

`TypeVar()`表达式必须始终直接指定给变量（不应将其用作较大表达式的一部分）。 `TypeVar()`的参数必须是一个等于它所赋予的变量名称的字符串。类型变量不能重新定义。

`TypeVar`支持将参数类型约束为一组固定的可能类型（注意：这些类型不能通过类型变量进行参数化）。例如，我们可以定义一个范围超过`str`和`bytes`的类型变量。默认情况下，类型变量覆盖所有可能的类型。约束一个类型变量的例子：

```python
from typing import TypeVar

AnyStr = TypeVar('AnyStr', str, bytes)

def concat(x: AnyStr, y: AnyStr) -> AnyStr:
    return x + y
```

函数`concat`可以用两个`str`参数或两个`bytes`参数调用，但不能混合使用`str`和`bytes`参数。

如果有的话，至少应该有两个约束条件;指定单个约束是不允许的。

由类型变量约束的类型的子类型应该在类型变量的上下文中被视为它们各自明确列出的基类型。考虑这个例子：

```python
class MyStr(str): ...

x = concat(MyStr('apple'), MyStr('pie'))
```

该调用是有效的，但类型变量`AnyStr`将设置为`str`而不是`MyStr`。实际上，分配给`x`的返回值的推断类型也将是`str`。

另外，`Any`对于每个类型变量都是有效的值。考虑以下：

```python
def count_truthy(elements: List[Any]) -> int:
    return sum(1 for elem in elements if elem)
```

这相当于省略了通用符号并仅指出了`elements: List`。

### 用户定义的泛型类型（User-defined generic types）

您可以包含`Generic`基类，以将用户定义的类定义为通用类。例：

```python
from typing import TypeVar, Generic
from logging import Logger

T = TypeVar('T')

class LoggedVar(Generic[T]):
    def __init__(self, value: T, name: str, logger: Logger) -> None:
        self.name = name
        self.logger = logger
        self.value = value

    def set(self, new: T) -> None:
        self.log('Set ' + repr(self.value))
        self.value = new

    def get(self) -> T:
        self.log('Get ' + repr(self.value))
        return self.value

    def log(self, message: str) -> None:
        self.logger.info('{}: {}'.format(self.name, message))
```

作为基类的`Generic[T]`定义了类`LoggedVar`接受单个类型参数`T`.这也使得`T`作为类体内的类型有效。

`Generic`基类使用定义`__getitem__`的元类，以便`LoggedVar[t]`作为类型有效：

```python
from typing import Iterable

def zero_all_vars(vars: Iterable[LoggedVar[int]]) -> None:
    for var in vars:
        var.set(0)
```

泛型类型可以包含任意数量的类型变量，并且类型变量可能会受到限制。这是有效的：

```python3
from typing import TypeVar, Generic
...

T = TypeVar('T')
S = TypeVar('S')

class Pair(Generic[T, S]):
    ...
```

`Generic`的每个类型变量参数必须是不同的。因此这是无效的：

```python
from typing import TypeVar, Generic
...

T = TypeVar('T')

class Pair(Generic[T, T]):   # INVALID
    ...
```

`Generic[T]`基类在多种简单情况下是多余的，在这种情况下，您可以为其他泛型类生成子类并为其参数指定类型变量：

```python
from typing import TypeVar, Iterator

T = TypeVar('T')

class MyIter(Iterator[T]):
    ...
```

该类定义相当于：

```python
class MyIter(Iterator[T], Generic[T]):
    ...
```

您可以对`Generic`使用多重继承：

```python
from typing import TypeVar, Generic, Sized, Iterable, Container, Tuple

T = TypeVar('T')

class LinkedList(Sized, Generic[T]):
    ...

K = TypeVar('K')
V = TypeVar('V')

class MyMapping(Iterable[Tuple[K, V]],
                Container[Tuple[K, V]],
                Generic[K, V]):
    ...
```

在没有指定类型参数的情况下对泛型类进行子类化时，假定每个位置都有`Any`。在下面的例子中，`MyIterable`不是通用的，而是从`Iterable [Any]`隐式继承的：

```python
from typing import Iterable

class MyIterable(Iterable):  # Same as Iterable[Any]
    ...
```

通用元类不受支持。

### 类型变量的范围规则（Scoping rules for type variables）

类型变量遵循正常的名称解析规则。但是，在静态类型查询上下文中有一些特殊情况：

- 泛型函数中使用的类型变量可以被推断为代表相同代码块中的不同类型。例：

```python
from typing import TypeVar, Generic

T = TypeVar('T')

def fun_1(x: T) -> T: ... # T here
def fun_2(x: T) -> T: ... # and here could be different

fun_1(1)                  # This is OK, T is inferred to be int
fun_2('a')                # This is also OK, now T is str
```

- 在泛型类的方法中使用的类型变量与参数化此类的变量之一一致始终绑定到该变量。例：

```python
from typing import TypeVar, Generic

T = TypeVar('T')

class MyClass(Generic[T]):
    def meth_1(self, x: T) -> T: ... # T here
    def meth_2(self, x: T) -> T: ... # and here are always the same

a = MyClass() # type: MyClass[int]
a.meth_1(1)   # OK
a.meth_2('a') # This is an error!
```

- 在一个方法中使用的类型变量不匹配任何参数化该类的变量，这使得该方法成为该变量中的通用函数：

```python
T = TypeVar('T')
S = TypeVar('S')
class Foo(Generic[T]):
    def method(self, x: T, y: S) -> S:
        ...

x = Foo() # type: Foo[int]
y = x.method(0, "abc") # inferred type of y is str
```

- 未绑定的类型变量不应出现在泛型函数的主体中，也不应出现在方法定义以外的类体中：

```python
T = TypeVar('T')
S = TypeVar('S')

def a_fun(x: T) -> None:
    # this is OK
    y = [] # type: List[T]
    # but below is an error!
    y = [] # type: List[S]

class Bar(Generic[T]):
    # this is also an error
    an_attr = [] # type: List[S]

    def do_something(x: S) -> S: # this is OK though
        ...
```

- 泛型函数内出现的泛型类定义不应该使用泛型函数参数化的类型变量：

```python
from typing import List

def a_fun(x: T) -> None:

    # This is OK
    a_list = [] # type: List[T]
    ...

    # This is however illegal
    class MyGeneric(Generic[T]):
        ...
```

- 嵌套在另一个泛型类中的泛型类不能使用相同的类型变量。外部类的类型变量的范围不包括内部类：

```python
T = TypeVar('T')
S = TypeVar('S')

class Outer(Generic[T]):
    class Bad(Iterable[T]):      # Error
        ...
    class AlsoBad:
        x = None # type: List[T] # Also an error

    class Inner(Iterable[S]):    # OK
        ...
    attr = None # type: Inner[T] # Also OK
```

### 实例化泛型类和类型擦除（Instantiating generic classes and type erasure）

用户定义的泛型类可以被实例化。假设我们编写一个从`Generic[T]`继承的`Node`类：

```python
from typing import TypeVar, Generic

T = TypeVar('T')

class Node(Generic[T]):
    ...
```

要像创建一个普通类一样创建`Node`实例，请调用`Node()`。在运行时，实例的类型（类）将是`Node`。但是它对类型检查器有什么类型？答案取决于通话中有多少信息可用。如果构造函数（`__init__`或`__new__`）在其签名中使用`T`，并且传递了相应的参数值，则将替换相应参数的类型。否则，假设`Any`。例：

```python
from typing import TypeVar, Generic

T = TypeVar('T')

class Node(Generic[T]):
    x = None  # type: T  # Instance attribute (see below)
    def __init__(self, label: T = None) -> None:
        ...

x = Node('')  # Inferred type is Node[str]
y = Node(0)   # Inferred type is Node[int]
z = Node()    # Inferred type is Node[Any]
```

如果推断类型使用`[Any]`，但预期类型更具体，则可以使用类型注释（请参见下文）来强制变量的类型，例如：

```python
# (continued from previous example)
a = Node()  # type: Node[int]
b = Node()  # type: Node[str]
```

或者，您可以实例化特定的具体类型，例如：

```python
# (continued from previous example)
p = Node[int]()
q = Node[str]()
r = Node[int]('')  # Error
s = Node[str](0)   # Error
```

请注意，`p`和`q`的运行时类型（类）仍然只是`Node` - `Node[int]`和`Node[str]`是可区分的类对象，但通过实例化它们创建的对象的运行时类不记录区别。这种行为称为“类型擦除(type erasure)”;在泛型语言（例如Java，TypeScript）中是常见的做法。

使用泛型类（参数化或不参数化）来访问属性将导致类型检查失败。在类定义主体之外，不能分配类属性，只能通过不具有同名实例属性的类实例对其进行查找：

```python
# (continued from previous example)
Node[int].x = 1  # Error
Node[int].x      # Error
Node.x = 1       # Error
Node.x           # Error
type(p).x        # Error
p.x              # Ok (evaluates to None)
Node[int]().x    # Ok (evaluates to None)
p.x = 1          # Ok, but assigning to instance attribute
```

不能实例化通用版本的抽象集合，如`Mapping`或`Sequence`以及内置类的常规版本（`List`，`Dict`，`Set`和`FrozenSet`）。但是，可以实例化具体的用户定义的子类和具体集合的通用版本：

```python
data = DefaultDict[int, bytes]()
```

请注意，不应该混淆静态类型和运行时类。在这种情况下，类型仍然被删除，上面的表达式只是一个简写：

```python
data = collections.defaultdict()  # type: DefaultDict[int, bytes]
```

不建议直接在表达式中使用下标类（例如`Node[int]`） - 使用类型别名（例如`IntNode = Node[int]`）是首选。 （首先，创建下标类，例如`Node[int]`，具有运行时成本，其次，使用类型别名更具可读性。）

### 任意泛型类型作为基类（Arbitrary generic types as base classes）

`Generic[T]`仅作为基类有效 - 它不是一个合适的类型。但是，用户定义的泛型类型（例如上面示例中的`LinkedList[T]`和内置的泛型类型以及`List[T]`和`Iterable[T]`等ABCs）既可以作为类型也可以作为基类来使用。例如，我们可以定义一个专用于类型参数的`Dict`的子类：

```python
from typing import Dict, List, Optional

class Node:
    ...

class SymbolTable(Dict[str, List[Node]]):
    def push(self, name: str, node: Node) -> None:
        self.setdefault(name, []).append(node)

    def pop(self, name: str) -> Node:
        return self[name].pop()

    def lookup(self, name: str) -> Optional[Node]:
        nodes = self.get(name)
        if nodes:
            return nodes[-1]
        return None
```

`SymbolTable`是`dict`的子类和`Dict[str，List[Node]]`的子类型。

如果泛型基类有一个类型变量作为类型参数，这使得定义的类是通用的。例如，我们可以定义一个可迭代的泛型`LinkedList`类和一个容器：

```python
from typing import TypeVar, Iterable, Container

T = TypeVar('T')

class LinkedList(Iterable[T], Container[T]):
    ...
```

现在`LinkedList[int]`是一个有效的类型。请注意，我们可以在基类列表中多次使用`T`，只要我们不在`Generic[...]`内多次使用相同的类型变量`T`。

还要考虑下面的例子：

```python
from typing import TypeVar, Mapping

T = TypeVar('T')

class MyDict(Mapping[str, T]):
    ...
```

在这种情况下，`MyDict`有一个参数，`T`.

### 抽象的泛型类型（Abstract generic types）

`Generic`使用的元类是`abc.ABCMeta`的子类。通用类可以通过包含抽象方法或属性成为ABC，并且泛型类也可以具有ABCs作为基类而不存在元类冲突。

### 输入具有上限的变量（Type variables with an upper bound）

类型变量可以使用`bound = <type>`来指定上限（注意：<type>本身不能通过类型变量进行参数化）。这意味着对于类型变量替换（显式或隐式）的实际类型必须是边界类型的子类型。一个常见的例子是`Comparable`类型的定义，它可以很好地捕获最常见的错误：

```python
from typing import TypeVar

class Comparable(metaclass=ABCMeta):
    @abstractmethod
    def __lt__(self, other: Any) -> bool: ...
    ... # __gt__ etc. as well

CT = TypeVar('CT', bound=Comparable)

def min(x: CT, y: CT) -> CT:
    if x < y:
        return x
    else:
        return y

min(1, 2) # ok, return type int
min('x', 'y') # ok, return type str
```

（注意，这并不理想，例如`min('x'，1)`在运行时是无效的，但类型检查器只会推断返回类型为`Comparable`。不幸的是，解决这个问题需要引入更强大，更多复杂的概念，F-bound的多态性，我们今后可能会重新讨论这一点。）

上限不能与类型约束组合（如在所使用的`AnyStr`中，请参阅前面的示例）;类型约束会导致推断类型为_exactly_约束类型之一，而上限只需要实际类型是边界类型的子类型。

### 协变和逆变（Covariance and contravariance）

考虑一个带有子类`Manager`的类`Employee`。现在假设我们有一个带有`List[Employee]`注解参数的函数。我们是否应该允许使用`List[Manager]`类型的变量作为参数来调用该函数？许多人会在没有考虑后果的情况下回答“是的，当然”。但除非我们更多地了解函数，否则类型检查器应该拒绝这样的调用：该函数可能会将一个`Employee`实例附加到列表中，这将违反调用方中变量的类型。

事实证明，这样的论点是矛盾的，而直觉答案（在函数不改变它的论点的情况下是正确的）要求论证共变。有关这些概念的更长篇介绍可以在维基百科[[wiki-variance]](https://www.python.org/dev/peps/pep-0484/#wiki-variance)和[PEP 483](https://www.python.org/dev/peps/pep-0483)中找到;这里我们只是展示如何控制一个类型检查器的行为。

默认情况下，泛型类型在所有类型变量中都是不变的，这意味着使用`List[Employee]`类型注释的变量值必须与类型注释完全匹配 - 不允许类型参数的子类或超类（在本示例中为`Employee`）。

为了便于声明协变或逆变类型检查可接受的容器类型，类型变量接受关键字参数`covariant = True`或`contravariant = True`。其中至多有一个可能会通过。用这些变量定义的泛型被认为是对应变量的协变或逆变。按照惯例，建议使用以`_co`结尾的名称作为用`covariant = True`定义的类型变量，并且以`contravariant = True`定义的名称以`_contra`结尾。

一个典型的例子涉及定义一个不可变（或只读）的容器类：

```python
from typing import TypeVar, Generic, Iterable, Iterator

T_co = TypeVar('T_co', covariant=True)

class ImmutableList(Generic[T_co]):
    def __init__(self, items: Iterable[T_co]) -> None: ...
    def __iter__(self) -> Iterator[T_co]: ...
    ...

class Employee: ...

class Manager(Employee): ...

def dump_employees(emps: ImmutableList[Employee]) -> None:
    for emp in emps:
        ...

mgrs = ImmutableList([Manager()])  # type: ImmutableList[Manager]
dump_employees(mgrs)  # OK
```

键入的只读集合类在其类型变量（例如`Mapping`和`Sequence`）中都声明为协变。可变集合类（例如`MutableMapping`和`MutableSequence`）被声明为不变。逆变类型的一个例子是`Generator`类型，它在`send()`参数类型中是逆变的（见下文）。

注意：协变或逆变不是类型变量的属性，而是使用此变量定义的泛型类的属性。差异仅适用于泛型类型;泛型函数没有这个属性。后者应该只使用没有`covariant`或`contravariant`关键字参数的类型变量来定义。例如，下面的例子很好：

```python
from typing import TypeVar

class Employee: ...

class Manager(Employee): ...

E = TypeVar('E', bound=Employee)

def dump_employee(e: E) -> None: ...

dump_employee(Manager())  # OK
```

同时禁止以下内容：

```python
B_co = TypeVar('B_co', covariant=True)

def bad_func(x: B_co) -> B_co: # Flagged as error by a type checker
    ...
```

### 数字塔（The numeric tower）

[PEP 3141](https://www.python.org/dev/peps/pep-3141)定义了Python的数字塔，stdlib模块号实现了相应的ABCs（`Number`，`Complex`，`Real`，`Rational`和`Integral`）。这些ABCs存在一些问题，但内置的具体数字类`complex`，`float`和`int`无处不在（特别是后两种:-)。

而不是要求用户编写`import numbers`，然后使用`numbers.Float`等，这个PEP提出了一个几乎同样有效的简单快捷方式：当参数被注释为具有`float`类型时，`int`类型的参数是可接受的;类似的，对于标注为具有复杂类型的参数，类型为`float`或`int`的参数是可接受的。这不处理实现相应ABCs或`fractions.Fraction`类，但我们相信这些用例非常罕见。

### 转发引用（Forward references）

当一个类型提示包含尚未定义的名称时，该定义可以表示为一个字符串文字，稍后解析。

发生这种情况的情况通常是容器类的定义，其中定义的类出现在某些方法的签名中。例如，以下代码（简单二叉树实现的开始）不起作用：

```python
class Tree:
    def __init__(self, left: Tree, right: Tree):
        self.left = left
        self.right = right
```

为了解决这个问题，我们写道：

```python
class Tree:
    def __init__(self, left: 'Tree', right: 'Tree'):
        self.left = left
        self.right = right
```

字符串文字应该包含一个有效的Python表达式（即，`compile(lit, '', 'eval')`应该是一个有效的代码对象），一旦模块被完全加载，它应该没有错误地评估。在其中进行评估的本地和全局命名空间应该是相同的命名空间，其中将对同一个函数的默认参数进行评估。

此外，该表达式应该可以解析为有效的类型提示，即它受到上面的<a href="#可接受的类型提示（Acceptable-type-hints）">可接受类型提示</a>部分中的规则的约束。

可以将字符串文字用作类型提示的一部分，例如：

```python
class Tree:
    ...
    def leaves(self) -> List['Tree']:
        ...
```

前向引用的常用用途是当例如Django模型需要签名。通常情况下，每个模型都在一个单独的文件中，并且具有参数类型涉及其他模型的参数的方法。由于循环导入以Python工作的方式，通常不可能直接导入所有需要的模型：

```python
# File models/a.py
from models.b import B
class A(Model):
    def foo(self, b: B): ...

# File models/b.py
from models.a import A
class B(Model):
    def bar(self, a: A): ...

# File main.py
from models.a import A
from models.b import B
```

假设main首先被导入，这将会在`from models.a import A`时引发一个ImportError，因为`import A`在`models/b.py`中，这是在定义了类A之前从`models/a.py`导入的。解决方案是切换仅模块导入，并通过其`_module _._ class_ name`引用模型：

```python
# File models/a.py
from models import b
class A(Model):
    def foo(self, b: 'b.B'): ...

# File models/b.py
from models import a
class B(Model):
    def bar(self, a: 'a.A'): ...

# File main.py
from models.a import A
from models.b import B
```
