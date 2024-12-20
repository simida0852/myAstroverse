---
title: javaScript类(Class)完全指南
slug: javaScript类(Class)完全指南
pubDate: 2022-05-30
tags:
  - javaScript
  - 教程
description: javaScript类(Class)完全指南
author: xf
cover: src/images/js-1.webp
coverAlt: js
category:
  - js
---
## javaScript类(Class)完全指南

javaScript 使用原型继承:每个对象都从原型对象继承属性和方法。
在Java或Swift等语言中使用的传统类作为创建对象的蓝图，在 javaScript 中不存在，原型继承仅处理对象。
原型继承可以模拟经典类继承。为了将传统的类引入 javaScript, ES2015 标准引入了class语法，其底层实现还是基于原型，只是原型继承的语法糖。
这篇文章主要让你熟悉 javaScript 类:如何定义类，初始化实例，定义字段和方法，理解私有和公共字段，掌握静态字段和方法。
<!--truncate-->
## 1. 定义:类关键字

使用关键字class可以在 JS 中定义了一个类：

```javascript
class User {
    // 类的主体
}
```

上面的代码定义了一个User类。 大括号{}里面是类的主体。 此语法称为class 声明。
如果在定义类时没有指定类名。可以通过使用类表达式，将类分配给变量：

```javascript
const UserClass = class {
    // 类的主体
};
```

还可以轻松地将类导出为 ES6 模块的一部分，默认导出语法如下：

```javascript
export default class User {
    // 主体
}
```

命名导出如下：

```javascript
export class User {
    // 主体
}
```

当我们创建类的实例时，该类将变得非常有用。实例是包含类所描述的数据和行为的对象。
使用new运算符实例化该类，语法：instance = new Class()。
例如，可以使用new操作符实例化User类：

 ```javascript
const myUser = new User();
```

new User()创建User类的一个实例。

## 2. 初始化:constructor()

constructor(param1, param2, ...)是用于初始化实例的类主体中的一种特殊方法。 在这里可以设置字段的初始值或进行任何类型的对象设置。
在下面的示例中，构造函数设置字段name的初始值

```javascript
class User {
  constructor(name) {
    this.name = name;
  }
}
```

User的构造函数有一个参数 name，用于设置字段this.name的初始值
在构造函数中，this 值等于新创建的实例。用于实例化类的参数成为构造函数的参数：

```javascript
class User {
  constructor(name) {
    name; // => 'Fundebug'
    this.name = name;
  }
}

const user = new User("Fundebug");
```

构造函数中的name参数的值为'Fundebug'。如果没有定义该类的构造函数，则会创建一个默认的构造函数。默认的构造函数是一个空函数，它不修改实例。
同时，一个 javaScript 类最多可以有一个构造函数。

## 3.字段

类字段是保存信息的变量，字段可以附加到两个实体：

1. 类实例上的字段
2. 类本身的字段(也称为静态字段)

字段有两种级别可访问性：

1. public:该字段可以在任何地方访问
2. private:字段只能在类的主体中访问

#### 3.1 公共实例字段

让我们再次看看前面的代码片段:

```javascript
class User {
  constructor(name) {
    this.name = name;
  }
}
```

表达式this.name = name创建一个实例字段名，并为其分配一个初始值。然后，可以使用属性访问器访问name字段

```javascript
const user = new User("Fundebug");
user.name; // => 'Fundebug'
```

name是一个公共字段，因为你可以在User类主体之外访问它。
当字段在构造函数中隐式创建时，就像前面的场景一样，可能获取所有字段。必须从构造函数的代码中破译它们。
class fields proposal 提案允许我们在类的主体中定义字段，并且可以立即指定初始值：

```javascript
class SomeClass {
  field1;
  field2 = "Initial value";

  // ...
}
```

接着我们修改User类并声明一个公共字段name：

```javascript
class User {
  name;

  constructor(name) {
    this.name = name;
  }
}

const user = new User("Fundebug");
user.name; // => 'Fundebug'
```

name;在类的主体中声明一个公共字段name。
以这种方式声明的公共字段具有表现力：快速查看字段声明就足以了解类的数据结构，而且，类字段可以在声明时立即初始化。

```javascript
class User {
  name = "无名氏";

  constructor() {}
}

const user = new User();
user.name; // '无名氏'
```

类体内的name ='无名氏'声明一个字段名称，并使用值'无名氏'对其进行初始化。
对公共字段的访问或更新没有限制。可以读取构造函数、方法和类外部的公共字段并将其赋值。

#### 3.2 私有实例字段

封装是一个重要的概念，它允许我们隐藏类的内部细节。使用封装类只依赖类提供的公共接口，而不耦合类的实现细节。
当实现细节改变时，考虑到封装而组织的类更容易更新。
隐藏对象内部数据的一种好方法是使用私有字段。这些字段只能在它们所属的类中读取和更改。类的外部世界不能直接更改私有字段。
_私有字段只能在类的主体中访问。_
在字段名前面加上特殊的符号#使其成为私有的，例如#myField。每次处理字段时都必须保留前缀#声明它、读取它或修改它。
确保在实例初始化时可以一次设置字段#name：

```javascript
class User {
  #name;

  constructor (name) {
    this.#name = name;
  }

  getName() {
    return this.#name;
  }
}

const user = new User('Fundebug')
user.getName() // => 'Fundebug'

user.#name  // 抛出语法错误
```

# name是一个私有字段。可以在User内访问和修改#name。方法getName()可以访问私有字段#name

但是，如果我们试图在 User 主体之外访问私有字段#name，则会抛出一个语法错误:SyntaxError: Private field '#name' must be declared in an enclosing class。

#### 3.3 公共静态字段

我们还可以在类本身上定义字段:静态字段。这有助于定义类常量或存储特定于该类的信息。
要在 javaScript 类中创建静态字段，请使用特殊的关键字static后面跟字段名:static myStaticField
让我们添加一个表示用户类型的新字段type:admin或regular。静态字TYPE_ADMIN和TYPE_REGULAR是区分用户类型的常量:

```javascript
class User {
  static TYPE_ADMIN = "admin";
  static TYPE_REGULAR = "regular";

  name;
  type;

  constructor(name, type) {
    this.name = name;
    this.type = type;
  }
}

const admin = new User("Fundebug", User.TYPE_ADMIN);
admin.type === User.TYPE_ADMIN; // => true
```

static TYPE_ADMIN和static TYPE_REGULAR在User类内部定义了静态变量。 要访问静态字段，必须使用后跟字段名称的类：User.TYPE_ADMIN和User.TYPE_REGULAR。

#### 3.4 私有静态字段

有时，我们也想隐藏静态字段的实现细节，在时候，就可以将静态字段设为私有。
要使静态字段成为私有的，只要字段名前面加上#符号:static #myPrivateStaticField。
假设我们希望限制User类的实例数量。要隐藏实例限制的详细信息，可以创建私有静态字段：

```javascript
class User {
  static #MAX_INSTANCES = 2;
  static #instances = 0;

  name;

  constructor(name) {
    User.#instances++;
    if (User.#instances > User.#MAX_INSTANCES) {
      throw new Error("Unable to create User instance");
    }
    this.name = name;
  }
}

new User("张三");
new User("李四");
new User("王五"); // throws Error
```

静态字段User.#MAX_INSTANCES设置允许的最大实例数，而User.#instances静态字段则计算实际的实例数。
这些私有静态字段只能在User类中访问，类的外部都不会干扰限制机制：这就是封装的好处。

## 4.方法

字段保存数据，但是修改数据的能力是由属于类的一部分的特殊功能实现的：**方法**。
javaScript 类同时支持实例和静态方法。

#### 4.1 实例方法

实例方法可以访问和修改实例数据。实例方法可以调用其他实例方法，也可以调用任何静态方法。
例如，定义一个方法getName()，它返回User类中的name ：

 ```javascript
class User {
  name = "无名氏";

  constructor(name) {
    this.name = name;
  }

  getName() {
    return this.name;
  }
}

const user = new User("Fundebug");
user.getName(); // => 'Fundebug'
```

getName() { ... }是User类中的一个方法，getname()是一个方法调用:它执行方法并返回计算值(如果存在的话)。
在类方法和构造函数中，this值等于类实例。使用this来访问实例数据:this.field 或者调用其他方法:this.method()。
接着我们添加一个具有一个参数并调用另一种方法的新方法名称nameContains(str)：

```javascript
class User {
  name;

  constructor(name) {
    this.name = name;
  }

  getName() {
    return this.name;
  }

  nameContains(str) {
    return this.getName().includes(str);
  }
}

const user = new User("Fundebug");
user.nameContains("Fun"); // => true
user.nameContains("code"); // => false
```

nameContains(str) { ... }是User类的一种方法，它接受一个参数str。 不仅如此，它还执行实例this.getName()的方法来获取用户名。
方法也可以是私有的。 为了使方法私有前缀，名称以＃开头即可，如下所示：

```javascript
class User {
  #name;

  constructor(name) {
    this.#name = name;
  }

  #getName() {
    return this.#name;
  }

  nameContains(str) {
    return this.#getName().includes(str);
  }
}

const user = new User('Fundebug');
user.nameContains('Fun');   // => true
user.nameContains('Code'); // => false

user.#getName(); // SyntaxError is thrown
```

# getName()是一个私有方法。在方法nameContains(str)中，可以这样调用一个私有方法:this.#getName()

由于是私有的，#getName()不能在用User 类主体之外调用。

#### 4.2 getters 和 setters

getter和setter模仿常规字段，但是对如何访问和更改字段具有更多控制。在尝试获取字段值时执行getter，而在尝试设置值时使用setter。
为了确保User的name属性不能为空，我们将私有字段#nameValue封装在getter和setter中:

```javascript
class User {
  #nameValue;

  constructor(name) {
    this.name = name;
  }

  get name() {
    return this.#nameValue;
  }

  set name(name) {
    if (name === "") {
      throw new Error(`name field of User cannot be empty`);
    }
    this.#nameValue = name;
  }
}

const user = new User("Fundebug");
user.name; // getter 被调用, => 'Fundebug'
user.name = "Code"; // setter 被调用

user.name = ""; // setter 抛出一个错误
```

get name() {...} 在访问user.name会被执行。而set name(name){…}在字段更新(user.name = 'Fundebug')时执行。如果新值是空字符串，setter将抛出错误。

#### 4.3 静态方法

静态方法是直接附加到类的函数，它们持有与类相关的逻辑，而不是类的实例。
要创建一个静态方法，请使用特殊的关键字static和一个常规的方法语法:static myStaticMethod() { ... }。
使用静态方法时，有两个简单的规则需要记住：

1. 静态方法可以访问静态字段。
2. 静态方法不能访问实例字段。

例如，创建一个静态方法来检测是否已经使用了具有特定名称的用户。

```javascript
class User {
  static #takenNames = [];

  static isNameTaken(name) {
    return User.#takenNames.includes(name);
  }

  name = "无名氏";

  constructor(name) {
    this.name = name;
    User.#takenNames.push(name);
  }
}

const user = new User("Fundebug");

User.isNameTaken("Fundebug"); // => true
User.isNameTaken("Code"); // => false
```

isNameTaken()是一个使用静态私有字段User的静态方法用于检查已取的名字。
静态方法可以是私有的:static #staticFunction() {...}。同样，它们遵循私有规则:只能在类主体中调用私有静态方法。

## 5. 继承: extends

javaScript 中的类使用extends关键字支持单继承。
在class Child extends Parent { }表达式中，Child类从Parent继承构造函数，字段和方法。
例如，我们创建一个新的子类ContentWriter来继承父类User。

```javascript
class User {
  name;

  constructor(name) {
    this.name = name;
  }

  getName() {
    return this.name;
  }
}

class ContentWriter extends User {
  posts = [];
}

const writer = new ContentWriter("John Smith");

writer.name; // => 'John Smith'
writer.getName(); // => 'John Smith'
writer.posts; // => []
```

ContentWriter继承了User的构造函数，方法getName()和字段name。同样，ContentWriter类声明了一个新的字段posts。
注意，父类的私有成员不会被子类继承。

#### 5.1 父构造函数：constructor()中的super()

如果希望在子类中调用父构造函数，则需要使用子构造函数中可用的super()特殊函数。
例如，让ContentWriter构造函数调用User的父构造函数，以及初始化posts字段

```javascript
class User {
  name;

  constructor(name) {
    this.name = name;
  }

  getName() {
    return this.name;
  }
}

class ContentWriter extends User {
  posts = [];

  constructor(name, posts) {
    super(name);
    this.posts = posts;
  }
}

const writer = new ContentWriter("Fundebug", ["Why I like JS"]);
writer.name; // => 'Fundebug'
writer.posts; // => ['Why I like JS']
```

子类ContentWriter中的super(name)执行父类User的构造函数。
**注意**，在使用this关键字之前，**必须在子构造函数中执行super()**。调用super()确保父构造函数初始化实例。

 ```javascript
class Child extends Parent {
  constructor(value1, value2) {
    //无法工作
    this.prop2 = value2;
    super(value1);
  }
}
```

#### 5.2 父实例:方法中的super

如果希望在子方法中访问父方法，可以使用特殊的快捷方式super。

```javascript
class User {
  name;

  constructor(name) {
    this.name = name;
  }

  getName() {
    return this.name;
  }
}

class ContentWriter extends User {
  posts = [];

  constructor(name, posts) {
    super(name);
    this.posts = posts;
  }

  getName() {
    const name = super.getName();
    if (name === "") {
      return "无名氏";
    }
    return name;
  }
}

const writer = new ContentWriter("Fundebug", ["Why I like JS"]);
writer.getName(); // => '无名氏'
```

子类ContentWriter的getName()直接从父类User访问方法super.getName()，这个特性称为方法重写。
注意，也可以在静态方法中使用super来访问父类的静态方法。

## 6.对象类型检查：instanceof

object instanceof Class是确定object 是否为Class实例的运算符，来看看示例：

```javascript
class User {
  name;

  constructor(name) {
    this.name = name;
  }

  getName() {
    return this.name;
  }
}

const user = new User("Fundebug");
const obj = {};

user instanceof User; // => true
obj instanceof User; // => false
```

user是User类的一个实例，user instanceof User的计算结果为true。
空对象{}不是User的实例，相应地obj instanceof User为false。
instanceof是多态的:操作符检测作为父类实例的子类。

```javascript
class User {
  name;

  constructor(name) {
    this.name = name;
  }

  getName() {
    return this.name;
  }
}

class ContentWriter extends User {
  posts = [];

  constructor(name, posts) {
    super(name);
    this.posts = posts;
  }
}

const writer = new ContentWriter("Fundebug", ["Why I like JS"]);

writer instanceof ContentWriter; // => true
writer instanceof User; // => true
```

writer是子类ContentWriter的一个实例。运算符writer instanceof ContentWriter的计算结果为true。
同时ContentWriter是User的子类。因此writer instanceof User结果也为true。
如果想确定实例的确切类，该怎么办?可以使用构造函数属性并直接与类进行比较

```javascript
writer.constructor === ContentWriter; // => true
writer.constructor === User; // => false
```

## 7. 类和原型

必须说 JS 中的类语法在从原型继承中抽象方面做得很好。但是，类是在原型继承的基础上构建的。每个类都是一个函数，并在作为构造函数调用时创建一个实例。
以下两个代码段是等价的。
**类版本：**

```javascript
class User {
    constructor(name) {
        this.name = name;
    }

    getName() {
        return this.name;
    }
}

const user = new User("Fundebug");

user.getName(); // => 'Fundebug'
user instanceof User; // => true
```

使用原型的版本：

```javascript
function User(name) {
  this.name = name;
}

User.prototype.getName = function() {
  return this.name;
};

const user = new User("Fundebug");

user.getName(); // => 'Fundebug'
user instanceof User; // => true
```

如果你熟悉Java或Swift语言的经典继承机制，则可以更轻松地使用类语法。

## 8. 类的可用性

这篇文章中的类的一些特性有些还在分布第三阶段的提案中。在2019年底，类的特性分为以下两部分：

- 公共和私有实例字段是[Class fields proposal](https://github.com/tc39/proposal-class-fields)建议的一部分
- 私有实例方法和访问器是[Class private methods proposal](https://github.com/tc39/proposal-private-methods)建议的一部分
- 其余部分为 ES6 标准的一部分。

## 9. 总结

javaScript 类用构造函数初始化实例，定义字段和方法。甚至可以使用static关键字在类本身上附加字段和方法。
继承是使用extends关键字实现的:可以轻松地从父类创建子类，super关键字用于从子类访问父类。
要利用封装，将字段和方法设为私有以隐藏类的内部细节，私有字段和方法名必须以#开头。
