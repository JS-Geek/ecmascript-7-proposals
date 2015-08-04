# Class and Property Decorators

## Summary
装饰器（Decorators）使得在声明类时注释（annotate）和修改类和属性成为可能。  
  
ES5对象字面量支持在value的位置使用任意表达式，而ES6的类只支持字面量函数作为value。装饰器通过一个声明式语法恢复了在声明类时运行代码的能力。

## Detailed Design
一个装饰器是：

* 一个表达式
* 求值产生一个函数
* 接受目标对象（target），名称（name）和属性描述符（property descriptor）作为参数
* 可选地返回一个安装在目标对象上的属性描述符

例如以下一个简单的类定义：

```javascript
class Person {
  name() { return `${this.first} ${this.last}`}
}
```

对这个类进行求值，将安装一个`name`函数到`Person.prototype`，大致类似以下代码：

```javascript
Object.defineProperty(Person.prototype, 'name', {
  value: specifiedFunction,
  enumerable: false,
  configurable: true,
  writable: true
});
```

一个装饰器可以前置在定义属性的语法前：

```javascript
class Person {
  @readonly
  name() { return `${this.first} ${this.last}` }
}
```

现在，在安装描述符到`Person.prototype`前，执行引擎会先调用描述符：

```javascript
let descriptor = {
  value: specifiedFunction,
  enumerable: false,
  configurable: true,
  writable: true
};

descriptor = readonly(Person.prototype, 'name', descriptor) || descriptor;
Object.defineProperty(Person.prototype, 'name', descriptor);
```

装饰器函数的签名和`Object.defineProperty`一样，并且提供了在相关的`defineProperty`实际运行前进行调整的机会。  


前置于getter和/或setter前的装饰器会操作访问器描述符：

```javascript
class Person {
  @nonenumerable
  get kidCount() { return this.children.length; }
}

function nonenumerable(target, name, descriptor) {
  descriptor.enumerable = false;
  return descriptor;
}
```

一个更详细的例子展示了一个简单的装饰器如何内存化一个访问器：

```javascript
class Person {
  @memoize
  get name() { return `${this.first} ${this.last}` }
  set name(value) {
    let [first, last] = value.split(' ');
    this.first = first;
    this.last = last;
  }
}

let memoized = new WeakMap();
function memoize(target, name, descriptor) {
  let getter = descriptor.get, setter = descriptor.set;
  
  descriptor.get = function() {
    let table = memoizationFor(this);
    if(name in table) {
      return table[name];
    }
    return table[name = getter.call(this)];
  };
  
  descriptor.set = function(value) {
    let table = memoizationFor(this);
    setter.call(this, value);
    table[name] = value;
  };
}

function memoization(obj) {
  let table = memoized.get(obj);
  if(!table) {
    table = Object.create(null);
    memoized.set(obj, table);
  }
  return table;
}
```

也可以直接装饰类本身。此时，装饰器使用目标的构造器作为参数。

```javascript
// A simple decorator
@annotation
class MyClass { }

function annotation(target) {
   // Add a property on target
   target.annotated = true;
}
```

因为装饰器是一个表达式，所以也可以像工厂方法一样获取额外的参数。

```javascript
@isTestable(true)
class MyClass { }

function isTestable(value) {
   return function decorator(target) {
      target.isTestable = value;
   }
}
```

同样的技术也可以用于属性装饰器：

```javascript
class C {
  @enumerable(false)
  method() { }
}

function enumerable(value) {
  return function (target, key, descriptor) {
     descriptor.enumerable = value;
     return descriptor;
  }
}
```

因为描述符装饰器操作目标对象，所以它们自然也可以作用在静态方法上。唯一的区别是装饰器的第一个参数是类本身（类构造器）而不是prototype，因为class本身是最初的`Object.defineProperty`的目标对象。  

基于同样的原因，描述符装饰器也可以作用在对象字面量上，把该对象作为target参数传给装饰器函数。

## Desugaring
### Class Declaration
#### Syntax
```javascript
@F("color")
@G
class Foo {
}
```

#### Desugaring (ES6)
```javascript
var Foo = (function(){
  class Foo {
  }
  
  Foo = F("color")(Foo = G(Foo) || Foo) || foo;
  return Foo;
})();
```

#### Desugaring (ES5)
```javascript
var Foo = (function(){
  function Foo() {
  }
  
  Foo = F("color")(Foo = G(Foo) || Foo) || foo;
  return Foo;
})
```

### Class Method Declaration
#### Syntax
```javascript
class Foo {
  @F("color")
  @G
  bar() {}
}
```

#### Desugaring (ES6)
```javascript
var Foo = (function() {
  class Foo {
   bar() {}
  }
  
  var _temp;
  _temp = F("color")(Foo.prototype, "bar", 
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || temp) || _temp;
  
  if(_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  
  return Foo;
})();

```

#### Desugaring (ES5)
```javascript
var Foo = (function() {
  function Foo() {
  }
  Foo.prototype.bar = function() { };
  
  var _temp;
  _temp = F("color")(Foo.prototype, "bar", 
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || temp) || _temp;
  
  if(_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  
  return Foo;
})();
```

### Class Accessor Declaration
#### Syntax
```javascript
class Foo {
  @F("color")
  @G
  get bar() {}
  set bar(value) { }
}
```

#### Desugaring (ES6)
```javascript
var Foo = (function() {
  class Foo {
    get bar() { }
    set bar(value) { }
  }
  
 var _temp;
  _temp = F("color")(Foo.prototype, "bar", 
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || temp) || _temp;
  
  if(_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  
  return Foo;
})();
```

#### Desugaring (ES5)
```javascript
var Foo = (function() {
  function Foo() { }
  
  Object.defineProperty(Foo.prototype, "bar", {
    get: function() { },
    set: function(value) { },
    enumerable: true, configurable: true
  });
  
 var _temp;
  _temp = F("color")(Foo.prototype, "bar", 
    _temp = G(Foo.prototype, "bar",
      _temp = Object.getOwnPropertyDescriptor(Foo.prototype, "bar")) || temp) || _temp;
  
  if(_temp) Object.defineProperty(Foo.prototype, "bar", _temp);
  
  return Foo;
})();
```

### Object Literal Method Declaration
#### Syntax
```javascript
var o = {
  @F("color")
  @G
  bar() { }
}
```

#### Desugaring (ES6)
```javascript
var o = (function() {
  var _obj = {
    bar() { }
  };
  
  var _temp;
  _temp = F("color")(_obj, "bar", 
    _temp = G(_obj, "bar",
      _temp = void 0) || temp) || _temp;
  
  if(_temp) Object.defineProperty(_obj, "bar", _temp);
  
  return _obj;
})();
```

#### Desugaring (ES5)
```javascript
var Foo = (function() {
  var _obj = {
    bar: function() { }
  };
  
  var _temp;
  _temp = F("color")(_obj, "bar", 
    _temp = G(_obj, "bar",
      _temp = void 0) || temp) || _temp;
  
  if(_temp) Object.defineProperty(_obj, "bar", _temp);
  
  return _obj;
})();
```

### Object Literal Accessor Declaration
#### Syntax
```javascript
var o = {
  @F("color")
  @G
  get bar() { }
  set bar(value) { }
}
```

#### Desugaring (ES6)
```javascript
var o = (function() {
  var _obj = {
    get bar() { }
    set bar(value) { }
  }
  
  var _temp;
  _temp = F("color")(_obj, "bar", 
    _temp = G(_obj, "bar",
      _temp = void 0) || temp) || _temp;
  
  if(_temp) Object.defineProperty(_obj, "bar", _temp);
  
  return _obj;
})();
```

#### Desugaring (ES5)
```javascript
var Foo = (function() {
  var _obj = {
  }
  Object.defineProperty(_obj, "bar", {
    get: function () { },
    set: function (value) { },
    enumerable: true, configurable: true
  });
  
  var _temp;
  _temp = F("color")(_obj, "bar", 
    _temp = G(_obj, "bar",
      _temp = void 0) || temp) || _temp;
  
  if(_temp) Object.defineProperty(_obj, "bar", _temp);
  
  return _obj;
})();
```
# Grammar

&emsp;&emsp;***DecoratorList***<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;&emsp;***DecoratorList***<sub> [?Yield]opt</sub>&emsp; ***Decorator***<sub> [?Yield]</sub>

&emsp;&emsp;***Decorator***<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;&emsp;`@`&emsp;***AssignmentExpression***<sub> [?Yield]</sub>

&emsp;&emsp;***PropertyDefinition***<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;&emsp;***IdentifierReference***<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;&emsp;***CoverInitializedName***<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;&emsp;***PropertyName***<sub> [?Yield]</sub>&emsp; `:`&emsp;***AssignmentExpression***<sub> [In, ?Yield]</sub>  
&emsp;&emsp;&emsp;&emsp;***DecoratorList***<sub> [?Yield]opt</sub>&emsp;***MethodDefinition***<sub> [?Yield]</sub>

&emsp;&emsp;***CoverMemberExpressionSquareBracketsAndComputedPropertyName***<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;&emsp;`[`&emsp;***Expression***<sub> [In, ?Yield]</sub>&emsp;`]`


> NOTE	产生式 *CoverMemberExpressionSquareBracketsAndComputedPropertyName* 被用来覆盖 *MemberExpression* 的解析， *MemberExpression* 是一个在 *ObjectLiteral* 或 *ClassBody* 内部的装饰器的一部分, 以此避免解析一个用于*ComputedPropertyName*的装饰器时的回看（loookahead）。


&emsp;&emsp;***PropertyName***<sub> [Yield, GeneratorParameter]</sub>&emsp;:  
&emsp;&emsp;&emsp;&emsp;***LiteralPropertyName***  
&emsp;&emsp;&emsp;&emsp;[+GeneratorParameter] ***CoverMemberExpressionSquareBracketsAndComputedPropertyName***  
&emsp;&emsp;&emsp;&emsp;[~GeneratorParameter] ***CoverMemberExpressionSquareBracketsAndComputedPropertyName***<sub> [?Yield]</sub>

&emsp;&emsp;***MemberExpression***<sub> [Yield]</sub>&emsp; :  
&emsp;&emsp;&emsp;&emsp;[Lexical goal ***InputElementRegExp***] ***PrimaryExpression***<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;&emsp;***MemberExpression***<sub> [?Yield]</sub>&emsp;***CoverMemberExpressionSquareBracketsAndComputedPropertyName***<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;&emsp;***MemberExpression***<sub> [?Yield]</sub>&emsp;`.`&emsp;***IdentifierName***  
&emsp;&emsp;&emsp;&emsp;***MemberExpression***<sub> [?Yield]</sub>&emsp;***TemplateLiteral***<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;&emsp;***SuperProperty***<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;&emsp;***NewSuper***&emsp;***Arguments***<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;&emsp;`new`&emsp;***MemberExpression***<sub> [?Yield]</sub>&emsp;***Arguments***<sub> [?Yield]</sub>

&emsp;&emsp;***SuperProperty***<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;&emsp;`super`&emsp;***CoverMemberExpressionSquareBracketsAndComputedPropertyName***<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;&emsp;`super`&emsp;`.`&emsp;***IdentifierName***

&emsp;&emsp;***ClassDeclaration***<sub> [Yield, Default]</sub>&emsp;:  
&emsp;&emsp;&emsp;&emsp;***DecoratorList***<sub> [?Yield]opt</sub>&emsp;`class`&emsp;***BindingIdentifier***<sub> [?Yield]</sub>&emsp;***ClassTail***<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;&emsp;[+Default] ***DecoratorList***<sub> [?Yield]opt</sub>&emsp;`class`&emsp;***ClassTail***<sub> [?Yield]</sub>

&emsp;&emsp;***ClassExpression***<sub> [Yield, GeneratorParameter]</sub>&emsp;:  
&emsp;&emsp;&emsp;&emsp;***DecoratorList***<sub> [?Yield]opt</sub>&emsp;`class`&emsp;***BindingIdentifier***<sub> [?Yield]opt</sub>&emsp;***ClassTail***<sub> [?Yield, ?GeneratorParameter]</sub>

&emsp;&emsp;***ClassElement***<sub> [Yield]</sub>&emsp;:  
&emsp;&emsp;&emsp;&emsp;***DecoratorList***<sub> [?Yield]opt</sub>&emsp;***MethodDefinition***<sub> [?Yield]</sub>  
&emsp;&emsp;&emsp;&emsp;***DecoratorList***<sub> [?Yield]opt</sub>&emsp;`static`&emsp;***MethodDefinition***<sub> [?Yield]</sub>

## Notes
为了更直接的支持元数据唯一（metadata-only）的装饰器，备受期望的静态分析特性，已经被TypeScript项目实现。它使得用户可以定义[ambient decorators](https://github.com/jonathandturner/brainstorming/blob/master/README.md#c6-ambient-decorators)，它支持一种更严格的语法，在不进行求值的情况下进行正确的分析。