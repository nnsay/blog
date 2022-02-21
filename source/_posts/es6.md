---
title: ES6 详解
date: 2019-10-20 22:14:45
tags: 
- JavaScript
- Code
- Reading
excerpt: ES6学习笔记
---

## 1. 块级作用域

`var 说明:` 

* var在方法中自动提升声明位置;

* for循环的时候引用共享的变量;

* var可以重复定义;

`新定义:`
<!-- more -->
* 使用块级作用域,块有{}和function定义;
* let const分别定义块级变量和常量;
* 块级变量仅在块作用域中生效, 且不能重复定义, 出了作用域无效;
* const定义变量不能重新赋值但值可变, 定义的时候必须初始化值;

## 2. 字符串和正则表达式

`正则表达式:`

* UTF-16的字符定义单元, 支持Unicode
* u: /./u, 不切割Unicode单元
* y: /./y, 严格模式, 新增正则表达式:sticky/lastIndex/
* RegExp新增第二个参数flags, 所以产生两个新属性: source/flags
`字符串模板:`

* 用\`包裹, 使用${}包裹本地变量;

* 字符串方法, eg: tag`${name} 123`

```

let count = 10,

    price = 0.25,

    message = tag`${count} items cost $${(count * price).toFixed(2)}`;



function tag(literals, ...substitutions) {

    let result = '';

    for(let i in substitutions) {

        result += literals.raw[i];

        result += substitutions[i].toString().repeat(2)

    }

    result += literals[literals.length -1];

    return result;

}

console.log(message);



```

## 3. 方法



` 默认值:`



* 方法参数可以设置默认值

* arguments仅保留最初方法执行时提供的参数列表和参数值

* 触发使用的参数默认值的条件相应位置上传递的对象是undefined或者不存在, 为null不能触发

* 多种默认值类型: 值类型, 对象, 方法, 表达式, 另一个已有的参数

```

function exec(name= 'jimmy', age=function(){return 10}, nickname=name) {

    age = 88; //arguments[1]的值仍然是10

   //...

}



exec('jimmy')//age使用默认值10

exec('jimmy', null)//age使用null作为其值

``

`剩余参数:`



* 使用...提供N个剩余参数, 在es5中可以使用arguments达到相同的效果

* ...的剩余参数比作为function.length的长度, 长度应该排除掉...类型

* ...剩余参数必须是方法的最后一个参数

es5:

function exec(object, k1,k2, k3...) {

    var klist = arguments.slice(1, arguments.length);

}



function exec(object, ...keys){

    var klist = keys;

}

```

`方法构造函数:`

* 动态创建方法, 最后一个参数是函数体,之前都是参数

* 支持剩余参数

* 支持无方法体

```

let exec = new Function('...args', 'return args.join("<>")');

let exec1 = new Function('name', 'nickname = name', 'return {name: name, nickname: nickname}');

console.log(exec(1, 2, 5)); //1<>2<>5 

console.log(exec1('jw')); //{ name: 'jw', nickname: 'jw' }

```



`延伸参数:`

* 使用...方式延伸N个参数

```

let values = [1, 2, 4, 5];

//es6, 混合延伸参数而正常的参数

console.log(Math.max(...values, 10));

//es5

console.log(Math.max.apply(null, values));

//common

console.log(Math.max(1, 2, 4, 5));

```

`方法名:`

* 提供一种方式获取方法定义的名称

```

function exec() { }

var exec1 = function () { }

var exec2 = new Function('return 2;');

console.log(exec.name); //exec

console.log(exec1.name); //exec1

console.log(exec2.name); //anonymous



var doSomething = function doSomethingElse () {};

var person = {

    get firstName() {

        return 'Jimmy'

    },

    sayName: function() {



    }

}

console.log(doSomething.name); //doSomethingElse

console.log(person.firstName.name); //undefined, 理论是get firstName, nodejs还没实现

console.log(person.sayName.name); //sayName

```



`new.target属性:`



* function有两个内置的属性[[constructor]] 和 [[call]], 如果使用new则new.target指向function实例, 否则是undefined

* 该属性仅限方法内部使用

```

function Person(name) {

    //if(this instanceof Person) { //es5

    if(new.target !== undefined) { //es6

        this.name = name;

    } else {

        throw new Error('use new to create Person');

    }

}



var person = new Person('jimmy');

console.log(person.name);

var noperson1 = Person('jimmy');

var noperson2 = Person.call(Person, 'jimmy');



```



`块级别的方法:`

* 定义在块级别的方法可以在其所在的块中访问, 但是出了块范围边不可访问;

* 使用匿名的块级方法,本质上是一个变量不是一个真正的第一公民;

* 不使用strict模式则块级别方法可以被访问;

```

'use strict';

if (true) {

    console.log(typeof doSomething); // 使用let 声明的function不会因为是function而被提升声明

    function doSomething() { }

    //let doSomething = function () { };

    doSomething();

}

console.log(typeof doSomething);



```



`箭头函数:`



使用()=>方式创建函数, 但是有许多限制:



* 没有 this, super, arguments, and new.target的绑定
* Cannot be called with new, 不能声明任何类型
* No prototype
* Can’t change this, 如果一个function包含一个箭头函数,这箭头函数使用这个function的this
* No arguments object
* No duplicate named parameters

* 支持bind,apply,call



```

//通常书写方式:

let exec = () => { };

console.log(exec.name)



//立即执行函数:

let person = ((name) => { return { getName: () => name } })('jimmy');

console.log(person.getName());//jimmy



//返回json, 使用小括号

const buildKey = id => ({ID: `CacheID:${id}`});



//仅有一个参数的时候可以简写如下:

const buildKey = id => { return `CacheID:${id}` };

console.log(buildKey(111));

//或者

const buildKey = (id) => `CacheID:${id}`;

//或者更简洁

const buildKey = id => `CacheID:${id}`;



```



`尾部调用优化:`



在一个function最后执行一个函数表达式, 如下:

```

function exec () {

    return doAnother();

}

```

常见的非尾部优化的写法:



> 最后调用的函数需要获取当前函数中的变量;



```

function exec() {

    var a = 10;

        process = ()=>{return a;}

    return process();

}



```

> 最后一个函数存储了之后再返回



```

function exec() {

    var result = ()=>{return 10;}

    return result;

}

```

> 最后一个函数表达式带有其他运算

```

function exec() {

    return doSomething() + 1;

}

```
 递归优化

```
//第二个return没有优化,导致上下文链很长
function factorial(n) {
    if (n <= 1) {
        return 1;
    } else {
        return n * factorial(n - 1);
    }
}
//优化之后
function factorialOptimized(n, p = 1) {
    if (n <= 1) {
        return p;
    } else {
        return factorialOptimized(n - 1, n * p);
    }
}
```
## 4. 扩展的对象方法

`对象字面量扩展:`

* 属性初始化:  如果属性名变量名相同, 可以直接使用本地变量, es会自动初始化
* 简明方法: 字面对象不提供一个属性为方法, 方法将会用其name属性分配同名属性
* 计算属性: 扩展es5中使用[]可以动态设置和获取对象属性,es6扩展[]为计算属性,带有[]的属性被理解表达式
```
//属性初始化:
let name = 'jimmy';
let exec = function () {
    return Promise.resolve(1);
}
let obj = {name, exec};
console.log(obj, obj.exec.name);//{ name: 'jimmy', exec: [Function: exec] } 'exec' 

//简明方法:
let person = {
    name: 'Jimmy',
   sayName () {}
};
console.log(person); //{ name: 'Jimmy', sayName: [Function: sayName] }

//计算属性:
let lastName = 'last name';
let suffix = ' name';
let person = {
    'first name': 'Jimmy',
    [lastName]: 'Wang',
    ['full' + suffix]: 'Jimmy Wang'
};
console.log(person);
```
`新方法:`
开发者经常在某些全局类型上加上自己的方法在全局使用, es6扩展了一些新的方法在适合的全局对象上

* Object.is(): 大多数情况等于===, 但是对于NaN, 正负0的比较有不同,Object.is会检查符号
* Object.assign(): 扩展对象, 第一个参数为接受对象, 当应用对象有访问属性, 该属性将转为数据属性在接受对象上

```
//is:
Object.is(0, -0);// false
Object.is('5', 5);//false
//assign:
Object.assign({c:3}, {a: 1}, {b:2}, {c:4});//{ c:4, a:1, b:2}, 有相同属性覆盖其值, 否则新建属性并赋值
```

`重复属性:`
es5中strict模式下字面对象如果有相同属性会报错, 但是在es6中strict模式允许重复属性

```
'use strict';
let person = {
    name: 'Jimmy',
    name: 'Wang'
};
console.log(person.name); //Wang
```
`获取所有属性名称并排序:`
使用Object.getOwnPropertyNames方法, 排序的顺序为: 数字为先升序, 字符串升序依次, 动态添加的在最后

```
var obj = {
    a: 1,
    0: 1,
    c: 1,
    2: 1,
    b: 1,
    1: 1
};
obj.d = 1; //动态添加
console.log(Object.getOwnPropertyNames(obj));// [ '0', '1', '2', 'a', 'c', 'b', 'd' ]
```
`原型扩展:`

* 使用Object.setPrototypeOf替换原型

```
let person = {
    getGreeting() {
        return "Hello";
    }
};
let dog = {
    getGreeting() {
        return "Woof";
    }
};
let friend = {
    getGreeting() {
        //return Object.getPrototypeOf(this).getGreeting.call(this) + ", hi!";
        return super.getGreeting() + ", hi!"; //优雅调用基类,等价于👆, super仅在简明方法时候可用
    }
};
// prototype is person
let _friend = Object.create(person);
console.log(_friend.getGreeting());
console.log(Object.getPrototypeOf(_friend) === person);  // true
// set prototype to dog
Object.setPrototypeOf(_friend, dog);
console.log(_friend.getGreeting());                      // "Woof"
console.log(Object.getPrototypeOf(_friend) === dog);     // true

// set prototype to person
Object.setPrototypeOf(friend, person);
console.log(friend.getGreeting());                      // "Hello, hi!"
console.log(Object.getPrototypeOf(friend) === person);  // true
// set prototype to dog
Object.setPrototypeOf(friend, dog);
console.log(friend.getGreeting());                      // "Woof, hi!"
console.log(Object.getPrototypeOf(friend) === dog);     // true

```
`正式方法定义:`
es6正式定义的方法还有内置的[[HomeObject]]属性,并且属于一个对象在创建的时候, 而不是直接创建, 通常两者没有区别, 但是使用super有区别, super执行的时候确定Prototype即调用Object.getPrototypeOf()获取[[HomeObject]]引用的环境

```
let person = {
    // method
    getGreeting() {
        return "Hello";
    }
};
// not a method
function shareGreeting() {
    return "Hi!";
}

// prototype is person
let friend = {
    getGreeting() {
        return super.getGreeting() + ", hi!";
    }
};
Object.setPrototypeOf(friend, person);
console.log(friend.getGreeting());  // "Hello, hi!"
```
## 5. 更简单的数据获取--解构
开发过程中经常需要获取对象的值进行分析, 如果对象特别复杂和大, 那么获取数据变得一件需要费精力的事情

`对象解构:`

* 解构分配: 解构对象并分配对应变量的值, 例如: let {a}=node
* 解构默认值: 解构时可以提供默认值, let {a=2} = node, 仅当解构的属性不存在才使用而不是判断其值
* 解构到不同变量: 解构时可以指定变量名称, let {a: name} = node
* 潜逃解构: 同普通解构一样, 但是需要提供解构路径, let { b:{a}} = node
```
//常规解构
let obj = {
    name: 'jimmy',
    age: 27,
    address: 'Beijing',
    birthday: new Date()
}
let {name, birthday} = obj;
console.log(name, birthday); //jimmy 2016-11-16T10:12:12.246Z

//解构分配
let node = {
    type: "Identifier",
    name: "foo"
},
type = "Literal",
name = 5;
// assign different values using destructuring, 正常{}作为块不能出现在等号左边所以用小括号
({ type, name } = node);
console.log(type);      // "Identifier"
console.log(name);      // "foo"

//解构默认值
let node = {
    type: "Identifier",
    name: "foo"
};
let { type, name, value = true } = node;
console.log(type);
console.log(name);
console.log(value);

//解构到不同变量
let node = {
    type: "Identifier",
    name: "foo"
};
let { type: localType, name: localName, address: city = 'Beijing' } = node;
console.log(localType);     // "Identifier"
console.log(localName);     // "foo"
console.log(city);     // "Beijing"

//嵌套解构
let node = {
    type: "Identifier",
    name: "foo",
    info: {
        address: undefined
    }
};
let { info: {address: city = 'Beijing'}} = node;
console.log(city);     // "Beijing"
```
`数组结构:`

对象可以结构,数组也可以但是使用中括号
* 解构分配: 解构对象重新赋值, 也可以用于交换位置
* 默认值: 解构时可以提供默认值
* 嵌套解构: 按照位置嵌套使用[]进行解构
* 剩余项: 使用..., 可以构造成复制数组
* 混合解构: 数组和对象混合而成的结构表达式,例如{[a]} = node
```
//解构分配
let colors = ["red", "green", "blue"],
    firstColor = "black",
    secondColor = "purple";
[firstColor, secondColor, ] = colors;
console.log(firstColor);        // "red"
console.log(secondColor);       // "green"

//交换位置
let a = 1,
    b = 2;
[a, b] = [b, a];
console.log(a);     // 2
console.log(b);     // 1

//默认值
let colors = [ "red" ];
let [ firstColor, secondColor = "green" ] = colors;
console.log(firstColor);        // "red"
console.log(secondColor);       // "green"

//嵌套解构
let colors = [ "red", [ "green", "lightgreen" ], "blue" ];
// later
let [ firstColor, [ secondColor ] ] = colors;
console.log(firstColor);        // "red"
console.log(secondColor);       // "green"

//剩余项
let colors = [ "red", "green", "blue" ];
let [ firstColor, ...restColors ] = colors;
console.log(firstColor);        // "red"
console.log(restColors.length); // 2
console.log(restColors[0]);     // "green"
console.log(restColors[1]);     // "blue"

//普通引用
let colors = [ "red", "green", "blue" ];
let all = [[0], colors];
all[1][0]=0;
console.log(all); //[ [ 0 ], [ 0, 'green', 'blue' ] ]
console.log(colors); //[ 'red', 'green', 'blue' ] 
//复制引用
let colors = [ "red", "green", "blue" ];
let all = [[0], colors.concat()]; //es5复制数组
all = [[0], ...colors)]; //es6复制数组
all[1][0]=0;
console.log(all); //[ [ 0 ], [ 0, 'green', 'blue' ] ]
console.log(colors); //[ 'red', 'green', 'blue' ] 
```

`解构参数:`
解构方法的参数
* 被解构的参数是必须的
* 直接在参数位置处解构或者在函数体内解构
* 可以提供默认值
```
function exec(name, age, info) {
    let {address, hobbies: [hobby], mobile = '1313131313'} = info;
    console.log(address, hobby, mobile);
    //Beijing Coding 1313131313
}
exec('jimmy', 27, { address: 'Beijing', hobbies: ['Coding'] })
```
## 6. Symbol类型
在es6增了一种原始类型Symbol

`创建Symbol:`
使用全局类型Symbol, 但是不能使用new, 可以创建一个私有属性, 无法通过常规方式获取
```
let firstNmae = Symbol();
let person = {};
person[firstNmae] = 'Jimmy';
console.log(person[firstNmae]);//Jimmy
```
`使用Symbol:`
配合Object.defineProperty或者Obejct.defineProperties,定义只读属性

```
let firstName = Symbol("first name");
// use a computed object literal property
let person = {
    [firstName]: "Nicholas"
};
// make the property read only
Object.defineProperty(person, firstName, { writable: false });
let lastName = Symbol("last name");
Object.defineProperties(person, {
    [lastName]: {
        value: "Zakas",
        writable: false
    }
});
//person[lastName] = 'new vlalue'; //error, Cannot assign to read only property
console.log(person[firstName]);
console.log(person[lastName]);
```
`分享Symbol:`
使用Symbol.for(key)获取之前定义的Symbol用来获取私有成员, Symbol.for(key)方法会根据传入的key, 返回相同的对应的Symbol, 可以使用Symbol.keyFor()获取一个Symbol的key,如果不是通过Symbol.for创建的则没有key返回undefined

* Symbol.for(key)
* Symbol.keyFor(symbol)

```
let uid = Symbol.for("uid");
let person = {
    [uid]: 'Jimmy'
};
let uid2 = Symbol.for('uid')
let uid3 = Symbol('uid');
console.log(Object.is(uid, uid2), uid === uid2);//true
console.log(person[uid2], person[uid]);//Jimmy Jimmy
console.log(Symbol.keyFor(uid3), Symbol.keyFor(uid));//undefined 'uid'
```

`强制转化:`
Symbol需要类型的显示强制转化, 例如String(symbol), 只能String强制转化.

```
let uid = Symbol.for('uid');
let desc = String(uid);

console.log(desc);
//console.log(uid + ''); //Cannot convert a Symbol value to a string
//console.log(uid + 1); //Cannot convert a Symbol value to a number
```

`返回Symbol属性:`
es6中Object.keys()和Object.getPropertyNames()不能返回Symbol属性, 需要将其放在对象中获取, 获取的方法是Obejct.getOwnPropertySymbols(object)

```
let uid = Symbol.for('uid');
let object = {
    [uid]: 12345
};
let symbols = Object.getOwnPropertySymbols(object);
console.log(symbols);//[ Symbol(uid) ] 
console.log(object[symbols[0]]);//12345
console.log(Object.keys(object), Object.getOwnPropertyNames(object)); //[] []
```
`暴露内部操作使用Symbol:`

* Symbol.hasInstance: 所有类型都继承了, 判断是否是类型实例
* Symbol.isConcatSpreadable: 结合数组的concat使用,标记对象可以被concat, 这个对象有用数字属性和length
* Symbol.match/Symbol.replace/Symbol.search/Symbol.split: 字符串和表达式,可以修改这些内部Symbol
* Symbol.toPrimitive: 所有对象继承,但是定义在原型链上, 原始化的时候会传递类型, 修改这个内置方法, 这是一个中间过程, 例如 a / 3, a先要原始化, 然后返回valueOf()或者toString()的值, 这个方法就是定义返回值之前的契机
* Symbol.toStringTag: 很多时候为了获取数据类型采用Object.prototype.toString.call()的方式, es6引入了该方式,并用该Symbol定义在对象原型链上, 帮助设置一个类型,另外可以重新定义toString()改写输出方法不影响Object.prototype.toString.call()的使用

```
//Symbol.hasInstance基本使用
console.log(Array[Symbol.hasInstance]([1]));//true
console.log(Object[Symbol.hasInstance]([]));//true
console.log(Date[Symbol.hasInstance](new Date()));//true
console.log(Date[Symbol.hasInstance](Date));//false

//Symbol.hasInstance高级使用
function SpecialNumber() {
}

Object.defineProperty(SpecialNumber, Symbol.hasInstance, {
    value: function (v) {
        return (v instanceof Number) && (v >= 1 && v <= 100);
    }
})
let one = new Number(1);
let zore = new Number(0);
console.log(zore instanceof SpecialNumber);
console.log(one instanceof SpecialNumber);

//Symbol.isConcatSpreadable
let collection = {
    0: "Hello",
    1: "world",
    length: 2,
    [Symbol.isConcatSpreadable]: true
};
let messages = [ "Hi" ].concat(collection);
console.log(messages.length);    // 3
console.log(messages);           // ["hi","Hello","world"]

//Symbol.match/Symbol.replace/Symbol.search/Symbol.split
let hasLengthOf10 = {
    [Symbol.match]: function (value) {
        return value.length === 10 ? [value.substring(0, 10)] : null;
    },
    [Symbol.replace]: function (value, replacement) {
        return value.length > 10 ? (replacement + value.substring(10)) : value;;
    },
    [Symbol.search]: function (value) {
        return value.length === 10 ? 0 : -1;
    },
    [Symbol.split]: function (value) {
        return value.length === 10 ? ["", ""] : [value];
    }
};
let message1 = "Hello world"; // 11 characters 
let message2 = "Hello John"; // 10 characters
let match1 = message1.match(hasLengthOf10);
let match2 = message2.match(hasLengthOf10);
console.log(match1); // null 
console.log(match2); // ["Hello John"]
let replace1 = message1.replace(hasLengthOf10, '*');
let replace2 = message2.replace(hasLengthOf10, '*');
console.log(replace1); // "*d" 
console.log(replace2); // "Hello John"
let search1 = message1.search(hasLengthOf10), search2 = message2.search(hasLengthOf10);
console.log(search1); // -1 
console.log(search2); // 0
let split1 = message1.split(hasLengthOf10), split2 = message2.split(hasLengthOf10);
console.log(split1); // ["Hello world"] 
console.log(split2); // ["", ""]

//Symbol.toPrimitive
function Temperature(degrees) {
    this.degrees = degrees;
}
Temperature.prototype[Symbol.toPrimitive] = function (hint) {
    switch (hint) {
        case "string":
            return this.degrees + "\u00b0"; // degrees symbol
        case "number":
            return this.degrees;
        case "default":
            return this.degrees + " degrees";
    }
};
var freezing = new Temperature(32);
console.log(freezing + "!");// "32 degrees!" 
console.log(freezing / 2);// 16
console.log(String(freezing));// "32°"

//Symbol.toStringTag
function Person(name) {
    this.name = name;
}
Person.prototype[Symbol.toStringTag] = "Person";
var me = new Person("Nicholas");
console.log(me.toString());// "[object Person]"
console.log(Object.prototype.toString.call(me));// "[object Person]"
```
### 7. set和map
在es6前开发者通常自定义出set用来去重一组数据,也会构造出map方便开发, 现在看看之前如何构造和运用的,

```
//set更重视对象是否在组合中,map更重视取值
let set = Object.create(null);
set.foo = true;
if(set.foo) {
    console.log('the foo in set!');
}
let map = Object.create(null);
map['name'] = 'Jimmy';
console.log(map['name']);

//会有问题在有些时候
let map = Object.create(null);
let k1 = {};
let k2 = {};
map[k1] = 'Jimmy';
console.log(map[k2]); //Jimmy, 这里没有设置分配和设置k2属性但是可以访问
```
`Set 类型:`

* 可以由数组生成: new Set([1,2,4])
* 去重
* 可以放多种类型的成员
* 引用成员变量, 成员变量释放但是Set中的引用成员不释放(strong set)
* 常用方法: add(), delete(), has(), clear(), forEach()
* 常用属性: size
* 转化为数组: set类型可以通过剩余参数转化为一般数组
* weak set: 相对于strong set, 当引用的变量释放后, 该set中的成员也对应释放, 但是仅对于object类型的成员,
      WeakSet类型的与Set其他不同的还有可用的函数比较少仅有add()/has()/delete(), 没有size属性, 没有clear()

```
'use strict';

let set = new Set([1,1,1,]);
set.add(2);
set.add(2);
set.add('3');
console.log(set.size);//3, 去重
console.log(set.has(3));//false
console.log(set.delete('3'));//true
set.clear();
console.log(set);//Set {}

//forEach Set类型value和key是一样的,但是数组就不一样了
let set = new Set([1,2,3,]);
console.log(set.size);//3
set.forEach((value, key, owrSet)=>{
    console.log(value === key);//true
    console.log(owrSet);//Set {1,2,3}
});

//set 转化为set
let set = new Set([1,2,3,]);
let list = [...set];
let list1 = Array(set);
let list2 = new Array(set);
console.log(list, list1, list2);//[ 1, 2, 3 ] [ Set { 1, 2, 3 } ] [ Set { 1, 2, 3 } ]

//weak set
let set = new WeakSet();
let obj = 10;
//set.add(num);//throw error: TypeError: Invalid value used in weak set
obj = {};
set.add(obj);
obj = {};
obj = null;
console.log(set.has(obj));//false
console.log(set.delete(obj));//false
console.log(set);//WeakSet {}
```
`Map&WeakMap:`
同set和WeakSet差不多, 是一种新的对象存放键值信息

```
//map使用
let map = new Map();
let str = 'name';
let age = 10;
let s = Symbol();
map.set(str, 'jimmy');
map.set(age, 'age');
map.set(s, 'Symbol');
map.set(10, 10);//覆盖age
console.log(map.size);//3
console.log(map.values());//MapIterator { 'jimmy', 'age', 'Symbol' }
console.log(map.keys());//MapIterator { 'name', 10, Symbol() }
console.log(map.entries());//MapIterator { 'name', 10, Symbol() }
console.log(map.set());//
console.log(map.has(s));//true
console.log(map.get(s));// Symbol
console.log(map.delete(s));//true
map.forEach(function(value, key, map) {
    console.log(value, key, map); //jimmy name Map { 'name' => 'jimmy', 10 => 10, undefined => undefined
    //因为删除了一个s, 所以多了一个s
});
map.clear();
console.log(map.size);//0

//weak map & map
let map = new Map()
let wmap = new WeakMap();
let name = {};
map.set(name, 'jimmy');
wmap.set(name, 'jimmy');
name = null;
console.log(map);//Map {{}=> 'jimmy'}
console.log(wmap);//WeakMap {}

//初始化map
let map = new Map([['name', 'jimmy'], ['age', 10]]);
console.log(map);//Map { 'name' => 'jimmy', 'age' => 10 }

//使用WeakMap做对象内部临时变量, 好处是对象属性消失WeakMap也变小, 对象实例消失则WeakMap内部变量也消失, 节省资源
let Person = (function() {
    let privateData = new WeakMap();
    function Person(name) {
        privateData.set(this, { name: name });
    }
    Person.prototype.getName = function() {
        return privateData.get(this).name;
    };
    return Person;
} ());
let p1 = new Person('p1');
let p2 = new Person('p2');
console.log(p1.getName(), p2.getName());//p1 p2
或者不用WeakMap, 但是此时就不太优雅, p1不存在时privateData还有其数据:
let Person = (function() {
    let privateData = {};
    let _id = 0;
    function Person(name) {
        Object.defineProperty(this, 'id', { value: _id++ });
        privateData[this.id] = { name: name };
    }
    Person.prototype.getName = function() {
        return privateData[this.id].name;
    };
    return Person;
} ());
let p1 = new Person('p1');
let p2 = new Person('p2');
console.log(p1.getName(), p2.getName(), p1);
```
### 8 迭代器和生成器

`Iterator:`

* for-of: 遍历数组中的值而非索引

```
for(let item of [1,2,3]) {
    console.log(item);//1/2/3
}
```
`Generator:`

* 定义: function* fnname() {}
* 生成器方法: next, 返回值{value: 1, done: false}

```
let values = [1, 2, 3];
let iterator = values[Symbol.iterator]();
console.log(iterator.next());//{ value: 1, done: false }
console.log(iterator.next());//{ value: 2, done: false }
console.log(iterator.next());//{ value: 3, done: false }
console.log(iterator.next());//{ value: undefined, done: true }
```
`Symbol.iterator:`

利用iterator内置标识可以判断对象是否可以迭代
```
//基本使用
function isIteratable(object) {
    return typeof object[Symbol.iterator] === 'function';
}
console.log(isIteratable([]));//true
console.log(isIteratable('32'));//true
console.log(isIteratable(new Map()));//true
console.log(isIteratable(new Set()));//true
console.log(isIteratable(new WeakMap()));//false
console.log(isIteratable(new WeakSet()));//false

//扩展自定义可迭代类型
//定义自定义的可迭代对象
let collection = {
    items: [],
    *[Symbol.iterator]() {
        for(let item of this.items) {
            yield item;
        }
        //上面的for of 等价于
        //yield *this.items;
    }
};
collection.items.push(1);
collection.items.push(2);
for(let x of collection) {
    console.log(x);//1/2
}
```

`内置迭代器:`
常见的Array, Set, Map,都具有以下三个方法, 需要注意的是Array中key是索引下标,Set中key和value相同,而Map中key和value就比较清晰了; 另外如果直接遍历可迭代对象而非使用以下三个方法的返回值其实是有默认行为的,
Array默认使用其values()结果遍历, Set和Array相同, 而Map默认使用其entries()的结果应对直接遍历

* entries(): 返回对象的key-value迭代器
* keys(): 返回对象的key迭代器
* values(): 返回对象的values迭代器

```
//map默认使用entries()的结果进行迭代
let data = new Map();
data.set("title", "Understanding ECMAScript 6");
data.set("format", "ebook");
// same as using data.entries()
for (let [key, value] of data) {
    console.log(key + "=" + value);
}
//title=Understanding ECMAScript 6
//format=ebook
```
`字符串迭代:`
字符串的双字节被当成两个分开的编码单元所以在不同遍历方式有所不同, 因为ES6支持Unicode所以新的迭代方法尝试解决字符串编码导致的这个问题, 从输出角度看for-of方法更合适
```
let message = "A 𠮷 B";

for (let i = 0; i < message.length; i++) {
    console.log(message[i]);
}
// A
//
// ��
//
// B
for (let i of message) {
    console.log(i);
}
// A
//
// 𠮷
//
// B
```
`NodeList迭代:`
前端HTML元素的迭代器
```
let divs = document.getElementByTagName('div');
for(let div of divs) {
    console.log(div.id);
}
```
`伸展操作和非数组迭代:`
* 使用...进行对象的伸展
* 使用...进行复制或者扩展数组
```
//使用伸展操作
let h = new Set(['swim', 'book', 'walk']);
let hl = [...h];
console.log(hl);//[ 'swim', 'book', 'walk' ]
let m = new Map([['name', 'Jimmy'], ['age', 100]]);
let ml = [...m];
console.log(ml);//[ [ 'name', 'Jimmy' ], [ 'age', 100 ] ]

//使用伸展操作进行数组操作
let a1 = [1, 2, 3];
let a2 = [100, 200, 300];
let a = [0, ...a1, ...a2];
a2 = null;
a1 = [];
console.log(a.length);//7
console.log(a);//[0,1,2,3,100,200,300]
```
`高级迭代方法:`

* generator的next传递参数: 当传递参数的时候表示上一个yield返回的值, 从执行位置来看下一个next的时候才执行上一个yield到这次yield中间的代码, 所以第一个next传参数不会生效

* generator的throw方法: 抛一个错误给generator终止generator
* generator的return表达式: 可以尽早的结束generator,不像throw,return表示正常的无错误结束即done:true
* for-of中的return: 经过测试for-of中的return终止迭代器,类似于break;
* 使用generator执行异步任务: 构造一个类似co的执行容器;

```
//next传递参数
function* f() {
    let first =  yield 1;
    let sencond = yield first + 4;
    yield sencond;
}
let g = f();
console.log(g.next());//{ value: 1, done: false }
console.log(g.next(4));//{ value: 8, done: false }
console.log(g.next());//{ value: undefined, done: false }

//throw抛出错误
function* f() {
    let first = yield 1;
    let sencond = yield first + 4;
    yield sencond;
}
let g = f();
console.log(g.next().value);//1
console.log(g.throw('error'));//trow error

//return表达式
function* f() {
    yield 1;
    return -1;
    yield 2;
}
let g = f();
console.log(g.next().value);//1
console.log(g.next());//{ value: -1, done: true }
console.log(g.next());//{ value: undefined, done: true }

//最简co
function run(gfn) {
    let result = gfn();
    let next = result.next();
    while (!next.done) {
        let value = next.value;
        next = result.next(value);
    }
}

run(function* () {
    let rst1 = yield 1;
    let rst2 = yield rst1 * 3;

    console.log('result: %d', rst2);//result: 3
});

//使用generator执行异步任务
function run2(gfn) {
    let result = gfn();
    let next = result.next();

    function processNext() {
        if (!next.done) {
            let value = next.value;

            value(function (err, rst) {
                console.log(err, rst);

                if (err) {
                    result.throw(err);
                }
                next = result.next(rst);
                processNext();
            });
        }
    }
    processNext();
}
function exec(number) {
    return function (callback) {
        setTimeout(function () {
            callback(null, number + 11);
        }, 50);
    };
}
run2(function* () {
    let rst = yield exec(100);
    console.log(rst); // 111
});
```
### 9 类
javascript中本来没有类的概念, 但是使用过程中开发者自己改造一些类的来实现自己的需求所以类被引入了es6, 类仅仅是es6的语法糖, 其本质还是还是基于function的, 类的关键词是:class

`为何使用类`

* class声明的类不像function不能被提升,程序只有运行到声明处才会生效;
* 所有在类内部的代码都会使用严格模式进行执行;
* 类的所有方法是不可枚举的;
* 所有的内部方法缺少[[constructor]]所以不能使用new执行, 使用new会报错;
* 调用类不是new会报错;
* 在类中试图修改类名会报错;

```
//类语法糖
class Person {
    constructor(name) {
        this.name = name;
    }
    sayName() {
        console.log(this.name);
    }
}
//等价的function实现
let Person1 = (function () { //不能被提升
    'use strict';//内部使用严格模式
    const Person2 = function (name) { //不能改名在内部
        if (typeof new.target === 'undefined') {//类必须使用new调用
            throw new Error('Constructor must be called with new.');
        }
        this.name = name;
    }
    Object.defineProperty(Person2.prototype, 'sayName', {
        value: function () {
            if (typeof new.target !== 'undefined') {//方法不能使用new调用
                throw new Error('Class function should be called without new.');
            }
            console.log(this.name);
        },
        enumerable: false,//方法不能枚举
        writable: true,
        configurable: true
    });

    return Person2;
}());
let person = new Person('Jimmy');
person.sayName();//Jimmy

console.log(person instanceof Person); //true
console.log(person instanceof Object); //true
console.log(typeof Person); //function
console.log(typeof Person.prototype.sayName);//function
```

`类的表达式:`
和function的表达式相似
```
//类的表达式
let Person = class {
    constructor(name) {
        this.name = name;
    }
    sayName() {
        console.log(this.name);
    }
};

let p = new Person('jimmy');
p.sayName();//'jimmy'
console.log(Person.name);//Person
```

`类的命名:`
如果是声明那么名称就确定了,如果是表达式那么类的名称取自接收表达式的变量名称;

```
let Person = class {
    constructor(name) {
        this.name = name;
    }
};
let Person2 = class Person3 {
    constructor(name) {
        this.name = name;
    }
};
class Person4 {

}
console.log(Person.name);//Person
console.log(Person2.name);//Person3
console.log(Person4.name);//Person4
console.log(typeof Person2);//function
console.log(typeof Person3);//undefined, class声明结束后let classname 就结束了被回收所以是undefined
```

`Class的使用:`

可以作为参数传递, 类表达式可以立即执行, 使用方法包装一个类型可以模拟第一等级的类
```
//类作为参数传递
function createObject(classDef) {
    return new classDef();
}
let person = createObject(class {
    sayTime() {
        console.log('%s', (new Date()).toString());
    }
});
person.sayTime(); //Wed Dec 21 2016 17:07:23 GMT+0800 (CST)

//立即调用
let person2 = new class {
    constructor(name) {
        this.name = name;
    }
    sayName() {
        console.log(this.name);
    }
}('Jimmy');

person2.sayName();//Jimmy
```
`类的存储器:`
可以使用关键字get/set 创建类私有成员存储器

```
//存储器
class CustomHtml {
    constructor(element) {
        this.element = element;
    }

    get html() {
        return this.element;
    }

    set html(value) {
        this.element = value;
    }
}
// direct equivalent to previous example
let CustomHTMLElement = (function () {
    "use strict";
    const CustomHTMLElement = function (element) {
        // make sure the function was called with new
        if (typeof new.target === "undefined") {
            throw new Error("Constructor must be called with new.");
        }
        this.element = element;
    }
    Object.defineProperty(CustomHTMLElement.prototype, "html", {
        enumerable: false,
        configurable: true,
        get: function () {
            return this.element.innerHTML;
        },
        set: function (value) {
            this.element.innerHTML = value;
        }
    });
    return CustomHTMLElement;
} ());
let descriptor = Object.getOwnPropertyDescriptor(CustomHtml.prototype, 'html');
console.log('get' in descriptor);//true
console.log('set' in descriptor);//true
console.log(descriptor.enumerable);//false
```
`计算成员名称:`
和对象字面量一样, 类也可以使用计算成员命名来设置

```
//计算成员名称
let methodName = "sayName";
class PersonClass {
    constructor(name) {
        this.name = name;
    }
    [methodName]() {
        console.log(this.name);
    }
};
let me = new PersonClass("Nicholas");
me.sayName();           // "Nicholas"

//存储器名称计算
let propertyName = "html";
class CustomHTMLElement {
    constructor(element = 'defalut') {
        this.element = element;
    }
    get [propertyName]() {
        return this.element;
    }
    set [propertyName](value) {
        this.element = value;
    }
}
let c = new CustomHTMLElement();
c.html = ('<a href="#">百度</a>');
console.log(c.html);//<a href="#">百度</a>
```
`Generator方法:`
类中创建生成器方法

```
//generator 方法
class MyClass {
    *createIterator() {
        yield 1;
        yield 2;
        yield 3;
    }
}
let instance = new MyClass();
let iterator = instance.createIterator();
console.log(iterator.next());//{ value: 1, done: false }

class Collecton {
    constructor() {
        this.items = [];
    }
    *[Symbol.iterator]() {
        yield* this.items;
    }
}

let c = new Collecton();
c.items.push(1);
c.items.push('2');
c.items.push(new Date());
for (let i of c) {
    console.log(i);
}
// 1
// 2
// 2016-12-21T10:40:32.248Z
```
`静态方法:`
静态方法处在function级别, 实例方法处在function的prototype级别, 静态方法不能被实例访问到.

```
//静态方法
function PersonType(name) {
    this.name = name;
}
// static method
PersonType.create = function (name) {
    return new PersonType(name);
};
// instance method
PersonType.prototype.sayName = function () {
    console.log(this.name);
};
//等价的类写法
class PersonClass {
    // equivalent of the PersonType constructor
    constructor(name) {
        this.name = name;
    }
    // equivalent of PersonType.prototype.sayName
    sayName() {
        console.log(this.name);
    }
    // equivalent of PersonType.create
    static create(name) {
        return new PersonClass(name);
    }
}
//静态方法使用类调用
let person = PersonType.create("jimmy");
person.sayName();//jimmy
let person2 = PersonClass.create("jimmy");
person.sayName();//jimmy

```

`继承:`
原生继承通过改写prototype完成, class继承通过extends关键字完成, 也是一种语法糖
* supper仅在extends的时候用, 其他场景报错;
* 继承时supper需要先执行, supper执行完后初始化this, 在此之前访问this报错;
* 避免调用supper的唯一方式是在constructor中返回一个对象;
```
//演示原生的继承
//原生继承
function Rectangle(length, width) {
    this.length = length;
    this.width = width;
}
Rectangle.prototype.getArea = function () {
    return this.length * this.width;
};
function Square(length) {
    Rectangle.call(this, length, length);
}
Square.prototype = Object.create(Rectangle.prototype, {
    constructor: {
        value: Square,
        enumerable: true,
        writable: true,
        configurable: true
    }
});
var square = new Square(3);
console.log(square.getArea());
console.log(square instanceof Square);
console.log(square instanceof Rectangle);
// 9
// true
// true

//es6 class的继承
class Rectangle {
    constructor(length, width) {
        this.length = length;
        this.width = width;
    }
    getArea() {
        return this.length * this.width;
    }
}
class Square extends Rectangle {
    constructor(length) {
        // equivalent of Rectangle.call(this, length, length)
        super(length, length);
    }
}
var square = new Square(3);
console.log(square.getArea());
console.log(square instanceof Square);
console.log(square instanceof Rectangle);
// 9
// true
// true

```
`影子方法:`
子类可以访问父类的方法, 子类也可以改写父类的同名方法,子类也可以灵活的选择在子类中任然调用父类的方法

```
class Square extends Rectangle {
    constructor(length) {
        super(length, length);
    }
    // override and shadow Rectangle.prototype.getArea()
    getArea() {
        return this.length * this.length;
    }
}
class Square extends Rectangle {
    constructor(length) {
        super(length, length);
    }
    // override, shadow, and call Rectangle.prototype.getArea()
    getArea() {
        return super.getArea();
    }
}
```

`继承静态方法:`
静态方法可以被派生类所继承

```
class Rectangle {
    constructor(length, width) {
        this.length = length;
        this.width = width;
    }
    getArea() {
        return this.length * this.width;
    }
    static create(length, width) {
        return new Rectangle(length, width);
    }
}
class Square extends Rectangle {
    constructor(length) {
        // equivalent of Rectangle.call(this, length, length)
        super(length, length);
    }
}
var rect = Square.create(3, 4); //创建的是Rectangle
console.log(rect instanceof Rectangle);// true
console.log(rect.getArea());// 12
console.log(rect instanceof Square);//false, 创建的是Rectangle不是Square
```
`从表达式派生类:`
在es6中可以从包含有[[Constructor]]方法的任何表达式中派生出新的类, 如果表达式是generator或者null则会报错, 必须保证表达式的结果是一个合法的类 

```
function Rectangle(length, width) {
    this.length = length;
    this.width = width;
}
Rectangle.prototype.getArea = function () {
    return this.length * this.width;
};
class Square extends Rectangle {
    constructor(length) {
        super(length, length);
    }
}
var x = new Square(3);
console.log(x.getArea());// 9
console.log(x instanceof Rectangle);// true

//动态的创建类
let SerializableMixin = {
    serialize() {
        return JSON.stringify(this);
    }
};
let AreaMixin = {
    getArea() {
        return this.length * this.width;
    }
};
function mixin(...mixins) {
    var base = function () { };
    Object.assign(base.prototype, ...mixins);
    //console.log(base.prototype);base {  getArea: [Function: getArea],  serialize: [Function: serialize] }
    return base;
}
class Square extends mixin(AreaMixin, SerializableMixin) {//动态派生
    constructor(length) {
        super();
        this.length = length;
        this.width = length;
    }
}
var x = new Square(3);
console.log(x.getArea());// 9
console.log(x.serialize());// "{"length":3,"width":3}"
```

`内置继承:`
直接继承现有类型, 获取其特性进行使用

```
// built-in array behavior
var colors = [];
colors[0] = "red";
console.log(colors.length);
colors.length = 0;// 1
console.log(colors[0]);// undefined
                   
// trying to inherit from array in ES5
function MyArray() {
    Array.apply(this, arguments);
}
MyArray.prototype = Object.create(Array.prototype, {
    constructor: {
        value: MyArray,
        writable: true,
        configurable: true,
        enumerable: true
} });
var colors = new MyArray();
colors[0] = "red";
console.log(colors.length);
colors.length = 0;// 0
console.log(colors[0]);// "red"

//trying to inherit from array in ES6
class MyArray extends Array {
    // empty
}
var colors = new MyArray();
colors[0] = "red";
console.log(colors.length);
colors.length = 0;// 1
console.log(colors[0]);// undefined
```

`Symbole.species:` 
支持物种标志的类型有以下几个,  通常定义为**静态**的**寄存属性**, 它返回一个constructor方法, 在方法内部创建实例的时候可以使用该constructor, 调用的方法为:this.constructor[Symbol.species]
* Array
* ArrayBuffer
* Map
* Set
* Promsie
* Typed array
```
class MyArray extends Array {
    // empty
}
let items = new MyArray(1, 2, 3, 4),
    subitems = items.slice(1, 3);
console.log(items instanceof MyArray);// true
console.log(subitems instanceof MyArray);// true

//动态使用当前类(物种)
class MyClass {
    //物种标志只能get
    static get [Symbol.species]() {
        return this;
    }
    constructor(value) {
        this.value = value;
    }
    clone() {
        return new this.constructor[Symbol.species](this.value);//this.constructor[Symbol.species]->this->MyClass
    }
}
class MyDerivedClass1 extends MyClass {
    // empty
}
class MyDerivedClass2 extends MyClass {
    static get [Symbol.species]() {//this.constructor[Symbol.species]->MyClass
        return MyClass;
    }
}
let instance1 = new MyDerivedClass1("foo"),
    clone1 = instance1.clone(),
    instance2 = new MyDerivedClass2("bar"),
    clone2 = instance2.clone();
console.log(clone1 instanceof MyClass);// true
console.log(clone1 instanceof MyDerivedClass1);// true
console.log(clone2 instanceof MyClass);// true
console.log(clone2 instanceof MyDerivedClass2);// false
```
`new.target获取类名:`
类中也可以获取类名进行相应的错误提示或者操作

```
// abstract base class
class Shape {
    constructor() {
        if (new.target === Shape) {
            throw new Error("This class cannot be instantiated directly.")
        }
 } }
class Rectangle extends Shape {
    constructor(length, width) {
        super();
        this.length = length;
        this.width = width;
} }

var y = new Rectangle(3, 4);
console.log(y instanceof Shape);// true
var x = new Shape();// throws an errorno error
```

### 10. 数组的提升

原生的Array在创建数组的时候回根据传入参数的不同表现出不一样的行为, 例如new Array(5)表示创建了一个长度是5的数组, 但是new Array('5')表示创建了一个数组,数组的第一项是字符串5, 同理new Array(1,2,4)等价于[1,2,3], 由于在使用上容易出错所以es6在数组方面有所优化和改进.



`创建数组:`



* Array.of(): 该方法只需要传递数组的项即可, 一个参数则是长度是1, N参数则长度是N; 

* Array.from: 从对象中创建数组,类似于Array.prototype.slice.call(), Array.from方提供了其他更方面的操作:

    1. 转化: from的第一个参数可以是要转化为数组的对象, 第二个参数可以传递对数组项的进一步转化, from还要第三个参数可以提供第二个方法的容器;

    2. from还支持具有Symbol.iterator标志方法的对象创建数组



```

//Array.of()

let items = Array.of(1,2);

console.log(items);//[1,2]

items = Array.of(2);

console.log(items);//[2]

items = Array([1,2]);

console.log(items);//[[1,2]]

items = Array(...[1,2]);

console.log(items);//[1,2]



//Array.from() === Array.prototype.slice.call()

let args = { 0: 'jimmy', 1: 'name', length: 2 };

console.log(Array.from(args));//['jimmy', 'name']

console.log(Array.prototype.slice.call(args));//['jimmy', 'name']

console.log(Array.from(args, value => `items: ${value}`));//[ 'items: jimmy', 'items: name' ]

let help = {

    diff: 1,

    add(v) {

        return v + this.diff;

    }

};

function translate() {

    return Array.from(arguments, help.add, help);

}

console.log(translate(1, 2, 3));//[2,3,4]

//从Symbol.iterator创建数组

let gen = {

    *[Symbol.iterator]() {

        yield* [1, 2, 3];

    }

};

console.log(Array.from(gen, help.add, help));//[2,3,4]

```



`新的数组方法:`



* find()/findIndex(): 从数组搜索符合条件第一个值或者第一个值的索引, 值没有返回undefined, 索引没有则返回-1;

* fill(): 填充数组,第一个参数是要填出的值, 第二个参数开始位置,第三个参数结束位置, 左闭右开替换,默认全部替换;

* copyWith(pasteIndex, copyIndex, copyLimit): 同时复制N个数组值覆盖改变相应的N个元素;



```

//find() findIndex()

let number = [25, 30, 35, 40, 45];

console.log(number.find(n => n));//25

console.log(number.find(n => n > 35));//45

console.log(number.find(n => n < 0));//undefined

console.log(number.findIndex(n => n));//0

console.log(number.findIndex(n => n > 35));//3

console.log(number.findIndex(n => n < 0));//-1



//fill(item, start, end)

let number = [25, 30, 35, 40, 45];

number.fill(1);

console.log(number);//[ 1, 1, 1, 1, 1 ]

number = [25, 30, 35, 40, 45];

number.fill(1, 3);

console.log(number);//[ 25, 30, 35, 1, 1 ];

number.fill(-1, 3, 4);//开始结束的区间是左闭右开,即4位置不做填充

console.log(number);//[25, 30, 35, -1, 1]

```



`Array Buffer:`

数组缓存存在于内存中, 是一切typed array的基础, typed array 是数字数组, 数字的长度不一样而已, 例如Int8和Int32. ArrayBuffer按照字节分配内存. Array Buffer长度在创建的时候一旦改变就无法更改, 可以通过slice进行截取, 但是原始的长度不能改变, 但是可以改变数组中不同索引对应的值. ArrayBuffer可以使用DataView来管理数组内数据, 管理方法如下:

* getInt8(byteOffset, littleEndian)

* setInt8(byteOffset, value, littleEndian)

* getUint8(byteOffset, littleEndian)

* getFloat32(byteOffset, littleEndian)

* setFloat32(byteOffset, value, littleEndian)

* getFloat64(byteOffset, littleEndian)

* setFloat64(byteOffset, value, littleEndian)

可以使用BYTES_PER_ELEMENT获取每个元素占用的字节数如:Int8Array.BYTES_PER_ELEMENT为1 或者(new Int32Array()). BYTES_PER_ELEMENT结果是4

```

let buffer = new ArrayBuffer(10);   // allocate 10 bytes

let buffer2 = buffer.slice(4, 6); //左闭右开区间截取数组

console.log(buffer.byteLength); //10

console.log(buffer2.byteLength);    // 2



let buffer = new ArrayBuffer(10);   // allocate 10 bytes

let buffer2 = buffer.slice(4, 6); //左闭右开区间截取数组

console.log(buffer.byteLength); //10

console.log(buffer2.byteLength);    // 2

let view = new DataView(buffer);

let view2 = new DataView(buffer, 5, 5);//从第五个开始截取5个元素

console.log(view, view2);

// DataView {

//   byteLength: 10,

//   byteOffset: 0,

//   buffer: ArrayBuffer { byteLength: 10 } }

// DataView {

//   byteLength: 5,

//   byteOffset: 5,

//   buffer: ArrayBuffer { byteLength: 10 } }



//读写

let buffer = new ArrayBuffer(2),

    view = new DataView(buffer);

view.setInt8(0, 5);

view.setInt8(1, -1);

console.log(view.getInt16(0));// 1535

console.log(view.getInt8(0));// 5

console.log(view.getInt8(1));// -1



//每个元素占用的元素

let ints = new Int8Array(5);

console.log(ints.BYTES_PER_ELEMENT);// 1



```

`TypedArray和普通Array相同点:`



* 都使用length获取长度, 但是TypedArray的length不可写;

* 都要相同方法, 但是TypedArray比普通的数组多了一个件事就是类型的检查, 如果map一个Int16Array生产一个新的数组那么这个数组还是Int16Array, 可以使用的方法如下:

copyWithin()

entries()

fill()

filter()

find()

findIndex()

forEach()

indexOf()

join()

keys()

lastIndexOf()

map()

reduce()

reduceRight()

reverse()

slice()

some()

sort()

values()
* 相同的迭代器

* 相同的of/from方法



`TypedArray和普通Array不同点:`



* TypedArray不是Array的实例

* 行为不同, TypedArray会检查类型, 类型非法的时候使用0

* 缺失的方法: concat() shift() pop() splice() push() unshift()

* 独有的方法: set() subarray()



```

//不是相同的类型

let ints = new Uint16Array([25, 50]);

console.log(ints instanceof Array);//false

console.log(Array.isArray(ints));//false



//行为不同

ints = new Int16Array(2);

ints[0] = 20;

ints[1] = 40;

console.log(ints.length);//2

ints[2] = 50;

console.log(ints.length);//2

console.log(ints[2]);//undefined



//检查类型,如果非法的时候使用0代替

ints = new Int16Array(['hi']);

console.log(ints.length);//1

console.log(ints[0]);//0



//独有方法set

let ints = new Uint16Array(4);

ints.set([1,2]);//从0开始赋值

ints.set([3,4],2);//从2开始赋值

console.log(ints.toString());//1,2,3,4



//独有方法subarray

let ints = new Int16Array([25, 50, 75, 100]),

    subints1 = ints.subarray(),//全部截取

    subints2 = ints.subarray(2),//从2开始截取

    subints3 = ints.subarray(1, 3);//从1开始截取到3截止左开右闭

console.log(subints1.toString());// 25,50,75,100

console.log(subints2.toString());// 75,100

console.log(subints3.toString());// 50,75



```



### 11 Promise和异步处理

为了解决回调深坑, Promise被引入了, Promise不是唯一的解决回调深坑的方法



`Proimise状态:`

promise的每一个then和catch都会创建一个job在eventloop中进行处理, 但是这些job结束语另一个job queue

* unsettled: appending

* seted: fulfilled/rejection



`创建unsettledPromise:`

使用Promise构造方法进行创建, 构造方法包含resolve和rejecte两个状态改变方法;Promise执行体是被立刻执行的,但是then是被异步触发的,可以看下面的例子;



```

let readFile = new Promise((resolve, reject) {

    //some process logic code

    resolve/reject

});



//Promise执行体同步执行

let promise = new Promise(function(resolve, reject) {

    console.log("Promise");

    resolve();

});

promise.then(function() { //then创建的job异步触发

    console.log("Resolved.");

});

console.log("Hi!");

// Promise

// Hi!

// Resolved.

```



`创建settledPromise:`

* Promise.resolve: 创建fulfilled状态的Promise;

* Promise.reject: 创建rejected状态的Promise;

* non-thenable

```

//resolve

let promise = Promise.resolve(42);

promise.then(function(value) {

    console.log(value);         // 42

});



//reject

promise = Promise.reject(42);

promise.catch(function(value) {

    console.log(value);         // 42

});


//non-thenable resolve
let exec = {

    then(resolve, reject) {

        resolve(42);

    }

};

let p1 = Promise.resolve(exec);

p1.then(value => console.log(value));



//non-thenable reject

let thenable = {

    then: function(resolve, reject) {

        reject(42);

    }

};

let p1 = Promise.resolve(thenable);

p1.catch(function(value) {

    console.log(value);     // 42

});



```



`执行错误:`



```

let exec = new Promise((resolve, reject) => { throw Error('some error'); });

exec.catch(err => console.log(err.message));//some error

exec = new Promise((resolve, reject) => { reject(Error('some error')); });

exec.catch(err => console.log(err.message));//some error

```

`全局Promise reject 处理:`

可以在任意Promsie使用then或者catch而不用在意Promise是否settled是一个有争议的话题, 因为这样就分辨不出来什么时候和位置处理逻辑被handle

* nodejs的rejecte handle事件:

    unhandledRejection: 在一次event loop中当promise被rejected但是没有handle的时候发送该事件;

    rejectionHandled: 在一次evet loop后当promise被rejected并且rejection handle被调用;

* 浏览器reject handle事件:

    unhandledrejection: 在一次event loop中当promise被rejected但是没有handle的时候发送该事件;

    rejectionhandled: 在一次evet loop后当promise被rejected并且rejection handle被调用;

```

let rejected = Promise.reject(42);

// at this point, rejected is unhandled

//rejected.then((rst) => console.log(rst));

rejected.catch(function (value) {

    // now rejected has been handled

    console.log(value);

});



//unhandledRejection

let rejected = Promise.reject(42);

process.on('unhandledRejection', function(reason, promise) {

    console.log(reason);//42

    console.log(promise);//Promise { <rejected> 42 }

});



//rejectionHandled

let rejected = Promise.reject(42);

setTimeout(() => {

    rejected.catch(err => console.log('error: %s', err));//error: 42

}, 1000);

process.on('rejectionHandled', function (promise) {

    console.log(promise);//Promise { <rejected> 42 }

});

// (node:77034) UnhandledPromiseRejectionWarning: Unhandled promise rejection (rejection id: 1): 42

// Promise { <rejected> 42 } or (node:77023) PromiseRejectionHandledWarning: Promise rejection was handled asynchronously (rejection id: 1)

// error: 42



//利用rejectedHandled和UnhandleRejection事件管理这些Promise

let possiblyUnhandledRejections = new Map();

// when a rejection is unhandled, add it to the map

process.on("unhandledRejection", function(reason, promise) {

    possiblyUnhandledRejections.set(promise, reason);

});

process.on("rejectionHandled", function(promise) {

    possiblyUnhandledRejections.delete(promise);

});

setInterval(function() {

    possiblyUnhandledRejections.forEach(function(reason, promise) {

        console.log(reason.message ? reason.message : reason);

        // do something to handle these rejections

        handleRejection(promise, reason);

    });

    possiblyUnhandledRejections.clear();

}, 60000);


//浏览的rejecte处理事件

let rejected;

window.onunhandledrejection = function (event) {

    console.log(event.type);// "unhandledrejection"

    console.log(event.reason.message);// "Explosion!"

    console.log(rejected === event.promise);// true

};

window.onrejectionhandled = function (event) {

    console.log(event.type);// "rejectionhandled"

    console.log(event.reason.message);// "Explosion!"

    console.log(rejected === event.promise);// true

};

rejected = Promise.reject(new Error("Explosion!"));

```

`Promise链:`

多个Promise通过then链接



`捕捉错误:`

在promise链中每个then都创建一个promise, 如果需要传递参数则return否则结束,执行顺序会按照链去执行,如果链中某个then发生错误则结束链执行执行catch的rejecte handle逻辑, 所以永远保证有一个catch方法在promise链的最后.



```

let p1 = new Promise(function (resolve, reject) {

    throw new Error("Explosion!");

});

p1.catch(function (error) {

    console.log(error.message);//Explosion!

    throw new Error("Boom!");

}).catch(function (error) {

    console.log(error.message);//Boom!

});

```



`返回参数在Promise链中:`

在Promise可以使用return返回一个参数p, 作用如使用类似于: return Promise.resolve(p), p代表参数.返回参数可以在then中使用可以在catch中使用.



```

//在then中传递参数

let p1 = new Promise(function(resolve, reject) {

    resolve(42);

 });

p1.then(function(value) {

    console.log(value);// "42"

    return value + 1;

}).then(function(value) {

    console.log(value);// "43"

});



//在catch中传递参数

let p1 = new Promise(function(resolve, reject) {

    reject(42);

 });

p1.catch(function(value) {

    console.log(value);// "42"

    return value + 1;

}).then(function(value) {

    console.log(value);// "43"

});


```



`返回Promise在Promise链中:`

可以在Promise链中返回一个Promise,理论上then本来就返回一个新的Promise所以在then或者catch中返回一个Promise很正常. 在fulfillment或者rejection中返回一个thenable对象被不能改变promise的执行顺序, 执行体按照顺序执行.



```

let p1 = new Promise(function(resolve, reject) {

    resolve(42);

});

let p2 = new Promise(function(resolve, reject) {

    resolve(43);

});

p1.then(function(value) {

    // first fulfillment handler

    console.log(value);     // 42

    return p2;

}).then(function(value) {

    // second fulfillment handler

    console.log(value);     // 43

});


```

`返回多个Promsie:`

使用Promise.all()或者Promise.race方法可以执行一组Promise的执行



* Promise.all(): 执行一组Promise当所有的子项都执行完成后返回结果数组, 如果其中有个发生错误马山终止执行, 返回一个错误;

* Promise.race(): 执行一组Promise只要有一个settled不管是fulfilled还是rejection都结束



```

let p1 = Promise.resolve(42);

let p2 = new Promise(function(resolve, reject) {

    resolve(43);

});

let p3 = new Promise(function(resolve, reject) {

    resolve(44);

});



let race = Promise.race([p1, p2, p3]);

let all = Promise.all([p1, p2, p3]);

race.then(function(value) {

    console.log(value); // 42

});

all.then(function(values) {

    console.log(values);//[ 42, 43, 44 ]

});

```



`基于Promise的派生:`

Promise和其他内置类型一样可以进行派生, 派生类会继承resolve和reject静态方法, 使用派生类生成的实例其类型是派生类而不是Promise.



```

class MyPromise extends Promise {

    // use default constructor

    success(resolve, reject) {

        return this.then(resolve, reject);

    }

    failure(reject) {

        return this.catch(reject);

    }

}

let promise = new MyPromise(function(resolve, reject) {

    resolve(42);

});

promise.success(function(value) {

    console.log(value); // 42

}).failure(function(value) {

    console.log(value);

});

```



`基于Promise的异步执行:`



```

'use strict';



let fs = require("fs");

function run(taskDef) {

    // create the iterator

    let task = taskDef();

    // start the task

    let result = task.next();

    // recursive function to iterate through

    (function step() {

        // if there's more to do

        if (!result.done) {

            // resolve to a promise to make it easy

            let promise = Promise.resolve(result.value);

            promise.then(function(value) {

                result = task.next(value);

                step();

            }).catch(function(error) {

                result = task.throw(error);

                step();

            });

        }

    } ());

}

// define a function to use with the task runner

function readFile(filename) {

    return new Promise(function(resolve, reject) {

        fs.readFile(filename, function(err, contents) {

            if (err) {

                reject(err);

            } else {

                resolve(contents);

            }

        });

    });

}

// run a task

run(function* () {

    let contents = yield readFile(".jshintrc");

    //doSomethingWith(contents);

    console.log("Done:" + contents);

});

```

### 12.代理和反射API

es6增添了一些很底层的JavaScript操作API, 在之前这些特性之后引擎可以做例如object.definedProperty(),

在开发过程中如果希望自己的类型可以具有默写内置类型的特性可以使用派生类, 但是很多细节还是需要注意的, 例如数组增加删除数组项的时候都必须更新length, 而length属性也影响数组的创建, 所以代理和反射被引入.代理Proxy使用new Proxy(target, {})的方式创建, target即为需要代理的目标, 代理可以阻截本应和引擎交互的API供开发者自定义使用, 默认代理全部对象特性. 而反射API是代理提供的默认的可以和引擎交互的的底层操作的集合:

| 代理方法  | 改写方法      | 默认行为|

|:----:|:----:|:----:|

| get | 读取属性 | Refect.get() |

| set | 写属性 | Reflect.set() |

| has | in操作 | Reflect.has() |

| deleteProperty | 删除属性 | Reflect.deleteProperty |

| getPropertyOf | Object.getPropertyOf() | Reflect.getPropertyOf() |

| setPropertyOf | Object.setPropertyOf() | Reflect.setPropertyOf() |

| isExtensiable | Object.isExtensiable() | Reflect.isExtensiable() |

| preventExtensions | Object.preventExtensions() | Reflect.preventExtensions() |

| getOwnPropertyDescriptor | Object.getOwnPropertyDescriptor | Reflect.getOwnPropertyDescriptor |

| defineProperty | Object.defineProperty() | Reflect.defineProperty() |

| ownKeys | Object.keys()&Object.getOwnPropertyNames()&Obejct.getOwnPropertySymbols() | Reflect.ownKeys()|

| apply | 调用一个方法 | Reflect.apply() |

| constrct | 使用new调用一个方法 | Reflect.construct() |

每一个代理方法改写对象的内置方法允许开发者打断和改写底层操作, 如果想用默认的内置方法可以调用对应的反射方法.



`创建简单的代理:`

使用 Proxy 进行创建. 



```

let target = {};

let proxy = new Proxy(target, {});



proxy.name = 'Jimmy';

console.log(target.name, proxy.name);//Jimmy, Jimmy



target.name = 'Wang';

console.log(target.name, proxy.name);//Wang, Wang

//代理本身不进行属性的存储, 实际的操作仍然在目标对象上

``` 



`通过使用set代理方法验证属性:`

通过设置一些代理方法的方法完成有意思的效果, 例如验证属性.首先了解set代理方法的参数:

* trapTarget: 代理方法代理的目标对象;

* key: 要去写的属性名称或者标识;

* value: 属性的值;

* receiver: 操作发生的对象(通常是代理);



```

let target = {

    name: 'Jimmy'

};

let proxy = new Proxy(target, {

    set(trapTarget, key, value, receiver) {

        //忽略已经存在的属性

        if (!trapTarget.hasOwnProperty(key)) {

            if (isNaN(value)) {

                throw new Error('该对象的属性值必须是Number类型');

            }

            //添加属性

            //return Reflect.set(trapTarget, key, value, receiver);

            return Reflect.set(...arguments);

        }

    }

});

proxy.age = 100;

console.log(target);

try {

    proxy.age1 = 'aa'; //{ name: 'Jimmy', age: 100 }

} catch (e) {

    console.log(e.message);//该对象的属性值必须是Number类型

}

```



`通过使用get代理方法实现对象形状验证:`

obejct shpes对象形状即为一组对象拥有的属性和方法的集合. get代理方法的参数:

* trapTarget: 代理目标对象;

* key: 目标属性名称或者标识;

* receiver: 操作发生的对象(通常是代理);



```

let target = {

    name: 'Jimmy'

};

let proxy = new Proxy(target, {

    get(trapTarget, key, receiver) {

        if (!(key in trapTarget)) {

            throw new Error('对象没有该属性');

        }

        return Reflect.get(trapTarget, key, receiver);

        //return Reflect.get(arguments);//这里不能使用arguments了因为内部处理了参数传递argument不在是传入的参数了

    }

});

console.log(proxy.name);//Jimmy

console.log(target.name);//Jimmy

try {

    console.log(proxy.age);

} catch (e) {

    console.log(e.message);//对象没有该属性

}

```

`通过has代理方法隐藏属性:`

has代理方法有两个参数:

* trapTarget: 代理目标对象;

* key: 属性名称或者标识;



```

let target = {

    name: "target",

    value: 42

};

let proxy = new Proxy(target, {

    has(trapTarget, key) {

        if (key === "value") {

            return false;

        } else {

            return Reflect.has(trapTarget, key);

        }

    }

});

console.log("value" in proxy);// false

console.log("name" in proxy);// true

console.log("toString" in proxy);// true

```



`通过deleteProperty代理方法禁止删除属性:`



```

//通过代理实现

let target = {

    name: 'Jimmy',

    age: 100

};

let proxy = new Proxy(target, {

    deleteProperty(trapTarget, key) {

        if (key === 'name') {

            throw new Error('name属性不可以被删除');

        }

        return Reflect.deleteProperty(...arguments);

    }

});

console.log(delete proxy.age);//true

try {

    delete proxy.name;

} catch (e) {

    console.log(e.message);//name属性不可以被删除

}

console.log(proxy, target);//{ name: 'Jimmy' } { name: 'Jimmy' }

//通过原生定义实现

let obj = {

    name: 'jimmy'

};

Object.defineProperty(obj, 'name', {

    configurable: false

});

console.log(delete obj.name);

```

`原型代理方法:`

getPrototypeOf和setPrototypeOf, Reflect的这个两个方法属于更底层的方法, 它们的执行在Obejct的同名方法之前, 而且Reflect不会做强迫转化所以限制比较多, 其中setProperty方法有两个参数:

* trapTarget: 代理目标对象;

* proto: 被作为原型的对象;

使用原型代理方法有几点限制:
* getPrototypeOf方法必须返回null或者一个对象,其他的返回值在运行时会报错, Object.getPrototypeOf()会一直检查所期望的值;
* setPrototypeOf在操纵不成功的时候必须返回false, 此时Object.setPrototypeOf抛错, 假如返回了其他非false的值视为成功;

```
//自定义
let target = {};
let proxy = new Proxy(target, {
    getPrototypeOf(trapTarget) {
        return null;
    },
    setPrototypeOf(trapTarget, proto) {
        return false;
    }
});
let targetProto = Object.getPrototypeOf(target);
let proxyProto = Object.getPrototypeOf(proxy);
console.log(targetProto === Object.prototype);//true
console.log(proxyProto === Object.prototype);//false
console.log(proxyProto);//null
// succeeds
Object.setPrototypeOf(target, {});
// throws an error
Object.setPrototypeOf(proxy, {});

//直接调动内置方法
let target = {};
let proxy = new Proxy(target, {
    getPrototypeOf(trapTarget) {
        return Reflect.getPrototypeOf(...arguments);
    },
    setPrototypeOf(trapTarget, proto) {
        return Reflect.setPrototypeOf(...arguments);;
    }
});
let targetProto = Object.getPrototypeOf(target);
let proxyProto = Object.getPrototypeOf(proxy);
console.log(targetProto === Object.prototype);//true
console.log(proxyProto === Object.prototype);//false
console.log(proxyProto);//{}
// succeeds
Object.setPrototypeOf(target, {});
// succeeds
Object.setPrototypeOf(proxy, {});
```

`对象扩展代理方法:`
相关的方法有两个preventExtensions()和isExtensiable(), 都仅接收一个参数即代理目标对象.这两个方法如果自定义那么对应的Reflect方法的返回值都是Boolean. 这里Reflect和Object的同名两个方法也相似, 但是不同, 同样Reflect更底层, 对于isExtensiable()来说如果传递不是对象Object正常判断是否扩展, 而Reflect报错, 因为Reflect级别的函数不做类型检查和转化, 如果参数是对象则相同结果; 对于preventExtensions()如果传递是参数是non-object则Reflect的方法会报错, 而Object的方法会返回其传递的参数, 如果传递的参数是对象则Reflect的方法返返回Boolean值.

```
//基础使用使用原始内置方法
let target = {};
let proxy = new Proxy(target, {
    isExtensible(trapTarget) {
        return Reflect.isExtensible(trapTarget);
    },
    preventExtensions(trapTarget) {
        return Reflect.preventExtensions(trapTarget);
    }
});
console.log(Object.isExtensible(target));// true
console.log(Object.isExtensible(proxy));// true
Object.preventExtensions(proxy);
console.log(Object.isExtensible(target));// false
console.log(Object.isExtensible(proxy));// false

//自定义禁止扩展
let target = {};
let proxy = new Proxy(target, {
    isExtensible(trapTarget) {
        return Reflect.isExtensible(trapTarget);
    },
    preventExtensions(trapTarget) {
        return false;//扩展失效
    }
});
console.log(Object.isExtensible(target));// true
console.log(Object.isExtensible(proxy));// true
try {
    Object.preventExtensions(proxy);
}
catch (e) {
    console.log(e.message);//'preventExtensions' on proxy: trap returned falsish
}
console.log(Object.isExtensible(target));// true
console.log(Object.isExtensible(proxy));// true
```

`属性描述器代理方法:`
相关的代理方法有defineProperty和getOwnPropertyDescriptor(), 代理方法提供阻断底层调用可以实现自定义.
其中defineProperty的三个参数分别是:
* trapTarget: 代理目标对象;
* key: 属性名称或者Symbol;
* descriptor: 属性名对应的描述器;
其中getOwnPropertyDescriptor的Reflect函数有严格的参数限制,返回值必须是null/undefined/object中的一个,如果是对象必须是enumerable/configurable/vlaue/writable/get/set, 若非满足这些条件则抛错.
Object和Reflect的defineProperty有不同, Object方法返回传入的第一个参数, 而Reflect返回true或者false
```
//基本使用方法
let target = {};
let proxy = new Proxy(target, {
    defineProperty(trapTarget, key, descriptor) {
        return Reflect.defineProperty(trapTarget, key, descriptor);
    },
    getOwnPropertyDescriptor(trapTarget, key) {
        return Reflect.getOwnPropertyDescriptor(trapTarget, key);
    }
});
Object.defineProperty(proxy, 'name', { configurable: true, enumerable: true, value: 'Jimmy' });
console.log(Object.getOwnPropertyDescriptor(proxy, 'name'), proxy.name);
// { value: 'Jimmy',
//   writable: false,
//   enumerable: true,
//   configurable: true 
// } 'Jimmy'

//阻止定义属性
let target = {};
let proxy = new Proxy(target, {
    defineProperty(trapTarget, key, descriptor) {
        if (typeof key === 'symbol') {
            return false;
        }
        return Reflect.defineProperty(trapTarget, key, descriptor);
    }
});
Object.defineProperty(proxy, 'name', { configurable: true, enumerable: true, value: 'Jimmy' });
console.log(proxy.name);
try {
    let age = Symbol.for('age');
    Object.defineProperty(proxy, age, { configurable: true, enumerable: true, value: 'Jimmy' });
} catch (e) {
    console.log(e.message);//'defineProperty' on proxy: trap returned falsish for property 'Symbol(age)'
}
console.log(proxy);//{ name: 'Jimmy' }

//定义属性描述限制
let proxy = new Proxy({}, {
    defineProperty(trapTarget, key, descriptor) {
        //Reflect将过滤掉非法的descriptor中设置项
        return Reflect.defineProperty(trapTarget, key, descriptor);
    }
});
Object.defineProperty(proxy, "name", {
    value: "proxy",
    enumerable: true,
    name: "custom"
});
let target = {};
Object.defineProperty(target, "name", {
    value: "proxy",
    enumerable: true,
    name: "custom"
});
console.log(proxy);//{ name: 'proxy' }, 因为name descriptor设置项非法
console.log(target);//{ name: 'proxy' }
```
`ownKeys代理方法:`
阻断内部[[OwnPerpertyKeys]]允许自定义, 默认返回Obejct.keys()/Object.getOwnPropetySymbols()/Object.getOwnPropertyNames()/Object.assign()的组合集合, ownKeys的参数仅有一个即代理目标对象, 返回值是数组, 可以对数据进行自定义的过滤. ownKeys可以返回一组属性的子集, 但是不营销Obejct.keys()和for-of的遍历,因为这些操作不调用代理, 但是for-in会收到营销.

```
let proxy = new Proxy({}, {
    ownKeys(trapTarget) {
        return Reflect.ownKeys(trapTarget).filter(key => {
            return typeof key !== "string" || key[0] !== "_";
        });
    }
});
let nameSymbol = Symbol("name");
proxy.name = "proxy";//√
proxy._name = "private";//x
proxy[nameSymbol] = "symbol";//√
let names = Object.getOwnPropertyNames(proxy),
    keys = Object.keys(proxy),
    symbols = Object.getOwnPropertySymbols(proxy);
console.log(names.length);// 1
console.log(names[0]);// "proxy"
console.log(keys.length);// 1
console.log(keys[0]);// "proxy"
console.log(symbols.length);// 1
console.log(symbols[0]);// "Symbol(name)"

```

`函数的代理:`
相关方法是apply()和construct(), 在所有的代理方法中仅有这两个的对象是方法, 其中apply的参数有:
* trapTarget: 代理目标方法;
* thisArgs: 函数在执行时this的值;
* argumentList: 被传入函数的参数数组;
construct的参数如下:
* trapTarget: 要执行的函数;
* argumentList: 被传入方法的参数数组;

```
//基本使用
let target = function () { return 42; },
    proxy = new Proxy(target, {
        apply: function (trapTarget, thisArg, argumentList) {
            return Reflect.apply(trapTarget, thisArg, argumentList);
        },
        construct: function (trapTarget, argumentList) {
            return Reflect.construct(trapTarget, argumentList);
        }
    });
// a proxy with a function as its target looks like a function
console.log(typeof proxy);// "function"
console.log(proxy());// 42
var instance = new proxy();
console.log(instance instanceof proxy);// true
console.log(instance instanceof target);// true

//函数参数验证和调用方式
let target = function (...values) { return values.reduce((pre, cur) => { return pre + cur; }, 0); },
    proxy = new Proxy(target, {
        apply: function (trapTarget, thisArg, argumentList) {
            argumentList.forEach((arg) => {
                if (typeof(arg) !== 'number') {
                    throw new Error('该函数仅接收Number参数');
                }
            });
            return Reflect.apply(trapTarget, thisArg, argumentList);
        },
        construct: function (trapTarget, argumentList) {
            throw new Error('该函数不能使用new');
        }
    });
console.log(proxy(1, 2, 3));//6
try {
    proxy(1, 2, '3');//throw error
} catch (e) {
    console.log(e.message);//该函数仅接收Number参数
}
try {
    new proxy(1, 2, '3');//throw error
} catch (e) {
    console.log(e.message);//该函数不能使用new
}

//单独控制函数调用方式, 当然也可以使用new.target在方法定义的时候检查
function Numbers(...values) {
    this.values = values;
}
let NumbersProxy = new Proxy(Numbers, {
    apply: function (trapTarget, thisArg, argumentList) {
        throw new TypeError("This function must be called with new.");
    },
    construct: function (trapTarget, argumentList) {
        argumentList.forEach((arg) => {
            if (typeof arg !== "number") {
                throw new TypeError("All arguments must be numbers.");
            }
        });
        return Reflect.construct(trapTarget, argumentList);
    }
});
let instance = new NumbersProxy(1, 2, 3, 4);
console.log(instance.values);               // [1,2,3,4]
// throws an error
try {
    NumbersProxy(1, 2, 3, 4);
} catch (e) {
    console.log(e.message);//This function must be called with new.
}

//抽象类代理
class AbstractNumbers {
    constructor(...values) {
        if (new.target === AbstractNumbers) { //只能使用new且继承自本身
            throw new TypeError("This function must be inherited from.");
        }
        this.values = values;
    }
}
let AbstractNumbersProxy = new Proxy(AbstractNumbers, {
    construct: function (trapTarget, argumentList) {
        return Reflect.construct(trapTarget, argumentList, function () { });
    }
});
let instance = new AbstractNumbersProxy(1, 2, 3, 4);
console.log(instance.values);// [1,2,3,4]

//不需要new执行类方法创建实例(奇葩改造)
function MyNumber(...values) {
    this.values = values;
}
class MyNumber1 {
    constructor(...values) {
        this.values = values;
    }
}

let proxy = new Proxy(MyNumber, {
    apply(trapTarget, thisArg, argumentList) {
        return new trapTarget(...argumentList);
    }
});

let n = proxy(1, 2, 3);
console.log(n instanceof MyNumber);
try {
    MyNumber1(1, 2, 3);
} catch (e) {
    console.log(e.message);//Class constructor MyNumber1 cannot be invoked without 'new'
}
try {
    MyNumber(1, 2, 3);
} catch (e) {
    console.log(e.message);//Cannot set property 'values' of undefined
}
```

`撤销代理:`
Proxy.revacable(), 其执行参数和创建一样, 其返回值包含两个对象:
* trapTarget: 代理目标的对象或者函数;
* revoke: 一个方法, 执行后则解绑代理关系;

```
function MyNumber(...values) {
    this.values = values;
}
let MyNumberProxy = new Proxy(MyNumber, {
    apply(trapTarget, thisArg, argumentList) {
        return new trapTarget(...argumentList);
    }
});
let {proxy, revoke} = Proxy.revocable(MyNumber, {
    apply(trapTarget, thisArg, argumentList) {
        return new trapTarget(...argumentList);
    }
});
console.log(proxy(1, 2, 3));//MyNumber { values: [ 1, 2, 3 ] }
revoke();
try {
    console.log(proxy(1, 2, 3));//throw error, 如果这里把proxy换成MyNumberProxy不会出错
} catch (e) {
    console.log(e.message);//Cannot perform 'apply' on a proxy that has been revoked
}
```

`数组问题解决:`
数组可以自动增长length而且可以使用length消减数组长度, 使用代理和反射解决类似数组的对象

```
function toUint32(value) {
    return Math.floor(Math.abs(Number(value))) % Math.pow(2, 32);
}
function isArrayIndex(key) {
    let numericKey = toUint32(key);
    return String(numericKey) == key && numericKey < (Math.pow(2, 32) - 1);
}
function createMyArray(length = 0) {
    return new Proxy({ length }, {
        set(trapTarget, key, value) {
            let currentLength = Reflect.get(trapTarget, "length");
            // the special case
            if (isArrayIndex(key)) {
                let numericKey = Number(key);
                if (numericKey >= currentLength) {
                    Reflect.set(trapTarget, "length", numericKey + 1);
                }
            } else if (key === "length") {
                if (value < currentLength) {
                    for (let index = currentLength - 1; index >= value;
                        index--) {
                        Reflect.deleteProperty(trapTarget, index);
                    }
                }
            }
            // always do this regardless of key type
            return Reflect.set(trapTarget, key, value);
        }
    });
}
let colors = createMyArray(3);
console.log(colors.length);// 3
colors[0] = "red";
colors[1] = "green";
colors[2] = "blue";
console.log(colors.length);// 3
colors[3] = "black";
console.log(colors.length);// 4
console.log(colors[3]);// "black"
```

`使用代理作为原型:`
原型可以作为原型使用, 但是使用了代理的特性, 底层不在调用部分Reflect API成了一个独立的对象.
作为原型时不能使用特性如: defineProperty()但是例如get()/set()/has()还是可以用的.
* get(): 仅当原型没有属性明时才执行ReflectAPI;
* has(): 仅当原型没有属性明时才执行ReflectAPI;
* set(): 仅当原型没有属性明时才执行ReflectAPI, 执行一次之后第二次变化不在触发;

```
//has, 仅当own没有给定的属性名称是调用
let target = {};
let thing = Object.create(new Proxy(target, {
    has(trapTarget, key) {
        return Reflect.has(trapTarget, key);
    }
}));
// triggers the `has` proxy trap
console.log("name" in thing);// false
thing.name = "thing";
// does not trigger the `has` proxy trap
console.log("name" in thing);// true
console.log(target);// {}

```

### 13. 使用模块封装代码
与之前大部分代码都被放入全局不同es6有了模块用来组织代码模块具有以下特点:
* 模块内部变量方法仅在模块内部生效即模块级别的作用域, 可以显示声明保留内部资源到外部
* 模块内部this的值是undefined
* 模块中不可以出现HTML组件解析代码

`基础的export:`
需要注意的是export的对象必须有name, 匿名对象必须使用module的default
```
// export data
export var color = "red";
export let name = "Nicholas";
export const magicNumber = 7;
// export function
export function sum(num1, num2) {
    return num1 + num1;
}
// export class
export class Rectangle {
    constructor(length, width) {
        this.length = length;
        this.width = width;
    }
}
// this function is private to the module
function subtract(num1, num2) {
    return num1 - num2;
}
// define a function...
function multiply(num1, num2) {
    return num1 * num2;
}
// ...and then export it later
export multiply;

//改名
function sum(num1, num2) {
    return num1 + num2;
}
export { sum as add };

//模块默认暴露对象
export default function (num1, num2) {
    return num1 + num2;
}
function sum(num1, num2) {
    return num1 + num2;
}
export default sum;

//通过改名设置默认曝露对象
function sum(num1, num2) {
    return num1 + num2;
}
export { sum as default };

```

`基本的import:`
import 的资源如使用const定义的变量不可以改变
```
//引入一个模块资源
import { multiply } from './module.js';

//引入多个
import { multiply, subtract } from './module.js';
console.log(multiply(3, 4));

//引入整个模块
import * as M from './module';
console.log(M.multiply(3, 4));

//改名
import { add as sum } from "./example.js";
console.log(typeof add);            // "undefined"
console.log(sum(1, 2));             // 3

//引入默认
// import the default
import sum from "./example.js";
console.log(sum(1, 2));     // 3

//同时引入默认和其他对象
import sum, { color } from "./example.js";
console.log(sum(1, 2));     // 3
console.log(color);         // "red"

//引入默认并改名
import { default as sum, color } from "./example.js";
```

`重新导出:`
将其他模块的已有对象再次从当前模块中导出.

```
//先引入在导出
import { sum } from "./example.js";
export { sum }
//直接导出
export { sum } from "./example.js";
//导出改名
export { sum as add } from "./example.js";
//导出所有其他模块
export * from "./example.js";
```
`引入模块但是不设置绑定:`
模块中可能修改了全局变量或者对象, 所以导入模块即应用了这些对全局对象的逻辑

```
// module code without exports or imports
Array.prototype.pushAll = function(items) {
    // items must be an array
    if (!Array.isArray(items)) {
        throw new TypeError("Argument must be an array.");
    }
    // use built-in push() and spread operator
    return this.push(...items);
};

import "./example.js";
let colors = ["red", "green", "blue"];
let items = [];
items.pushAll(colors);
```

### ES6小改变



 `Number方面:`

* Number.isInteger(): 判断一个数字是否是整型;

* Number.isSafeInteger(): 判断整型数字是否溢出;

* Number.MAX_SAFE_INTEGER: 最大合法整数;

* Number.MIN_SAFE_INTEGER: 最小合法整数;



`新的math方法:`

    略

`Unicode编码:`

可以使用Unicode定义参数

```

// valid in ECMAScript 5 and 6

var \u0061 = "abc";

console.log(\u0061);     // "abc"

// equivalent to:

console.log(a);          // "abc"

```

### ES7

es7引入三个新特性



* 5 **7

* Array.prototype.includes()

* function返回的严格模式



```

// syntax error

let result = -5 ** 2;

// okay

let result1 = -(5 ** 2);    // equal to -25

// also okay

let result2 = (-5) ** 2;    // equal to 25



let values = [1, NaN, 2];

console.log(values.indexOf(NaN));       // -1,  indexOf是比较值 NaN !== NaN

console.log(values.includes(NaN));      // true



values = [1, +0, 2];

console.log(values.indexOf(-0));        // 1

console.log(values.includes(-0));       // true

```
