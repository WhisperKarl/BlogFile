---
title: php基础语法知识
date: 2016-08-22 17:25:42
categories:
- 编程
tags:
- php
---
空闲时间学习学习php的知识，长期记录博客。知识点比较零散，只是对感觉有必要注意的知识点做个记录。
<!-- more -->

- 双引号和单引号的区别
双引号会解析变量，而单引号不会解析变量，单引号中的变量将会被当做字符串处理
- 较长字符串的处理
Heretic格式：我们可以使用Heredoc结构形式的方法来解决该问题，首先使用定界符表示字符串（<<<），接着在“<<<“之后提供一个标识符GOD，然后是字符串，最后以提供的这个标识符结束字符串。如：
```
$string1 = <<<GOD
我有一只小毛驴，我从来也不骑。
有一天我心血来潮，骑着去赶集。
我手里拿着小皮鞭，我心里正得意。
不知怎么哗啦啦啦啦，我摔了一身泥.
GOD;
echo $string1;
```

- NULL
NULL（NULL）：NULL是空类型，对大小写不敏感，NULL类型只有一个取值，表示一个变量没有值，当被赋值为NULL，或者尚未被赋值，或者被unset()，这三种情况下变量被认为为NULL。

- 常量的定义
`define("PI",3.14);`
判断常量是否被定义过用defiend()函数
- 赋值运算符
`=`和`&`:  `=`和其他语言一样。`&`类似于指针，指向同一个数据，公用一块内存，某一个变量的值发生变化时，另外一个值也会相应改变。
- 比较运算符
=== ： 全等，$a === $b ，如果a等于b它们的类型也相同，返回true
<> : 不等， 等同于!= ，值不等时返回true
!== ：非全等，$a!==$b，当a不等于b或者类型不同时，返回true
- 逻辑运算符

![逻辑运算符.png](http://occxq9xco.bkt.clouddn.com/1642800-f943c667e71ade49.png)
- 字符串连接运算符
"." 并不改变原字符串
".=" 改变原字符串，
`$a = $a."呵呵"; ` 等价于 `a.="呵呵"`
- 错误控制运算符
PHP中提供了一个错误控制运算符“@”，对于一些可能会在运行过程中出错的表达式时，我们不希望出错的时候给客户显示错误信息，这样对用户不友好。于是，可以将@放置在一个PHP表达式之前，该表达式可能产生的任何错误信息都被忽略掉；
如果激活了track_error（这个玩意在php.ini中设置）特性，表达式所产生的任何错误信息都被存放在变量$php_errormsg中，此变量在每次出错时都会被覆盖，所以如果想用它的话必须尽早检查。
需要注意的是：错误控制前缀“@”不会屏蔽解析错误的信息，不能把它放在函数或类的定义之前，也不能用于条件结构例如if和foreach等。
- foreach循环语句
常用语遍历数组，类似于OC中的 for-in

``` 
//不取下标的情况
<?php
$students = array(
'2010'=>'令狐冲',
'2011'=>'林平之',
'2012'=>'曲洋',
'2013'=>'任盈盈',
'2014'=>'向问天',
'2015'=>'任我行',
'2016'=>'冲虚',
'2017'=>'方正',
'2018'=>'岳不群',
'2019'=>'宁中则',
);//10个学生的学号和姓名，用数组存储

//使用循环结构遍历数组,获取学号和姓名  

foreach($students as $v){ 
echo $v;//输出（打印）姓名
echo "<br />";
}
?>
//取下标的情况
foreach($students as $key => $v){ 
echo $key.":".$v;//输出（打印）学号：姓名
echo "<br />";
}
```
- 数组赋值的三中方式
1)` $array[0] = "苹果"；`
2)`$arr = array("0"=>"苹果");`
3)`$arr = array("苹果");`
2和3等价
- 类和属性
在类中定义的变量称之为属性，通常属性跟数据库中的字段有一定的关联，因此也可以称作“字段”。属性声明是由关键字 public，protected 或者 private 开头，后面跟一个普通的变量声明来组成。属性的变量可以设置初始化的默认值，默认值必须是常量。
访问控制的关键字代表的意义为：
public：公开的
protected：受保护的
private：私有的
```
class Car {
//定义公共属性
public $name = '汽车';

//定义受保护的属性
protected $corlor = '白色';

//定义私有属性
private $price = '100000';
}
```
默认都为public，外部可以访问。一般通过->对象操作符来访问对象的属性或者方法，对于静态属性则使用::双冒号进行访问。当在类成员方法内部调用的时候，可以使用$this(类似于OC的self)伪变量调用当前对象的属性。
```
$car = new Car();
echo $car->name;   //调用对象的属性
echo $car->color;  //错误 受保护的属性不允许外部调用
echo $car->price;  //错误 私有属性不允许外部调用
```
受保护的属性与私有属性不允许外部调用，在类的成员方法内部是可以调用的。
```
class Car{
private $price = '1000';
public function getPrice() {
return $this->price; //内部访问私有属性
​    }
}
```
- 定义类的方法
方法就是在类中的function，很多时候我们分不清方法与函数有什么差别，在面向过程的程序设计中function叫做函数，在面向对象中function则被称之为方法。
同属性一样，类的方法也具有public，protected 以及 private 的访问控制。
访问控制的关键字代表的意义为：
public：公开的
protected：受保护的
private：私有的
我们可以这样定义方法：
```
class Car {
public function getName() {
return '汽车';
}
​}
$car = new Car();
echo $car->getName();
```
- static静态关键字
使用关键字static修饰的，称之为静态方法，静态方法不需要实例化对象，可以通过类名直接调用(类似于OC中的类方法)，操作符为双冒号::。
```
class Car {
public static function getName() {
return '汽车';
}
​}
echo Car::getName(); //结果为“汽车”
```
静态属性与方法可以在不实例化类的情况下调用，直接使用类名::方法名的方式进行调用。静态属性不允许对象使用->操作符调用。
```
class Car {
private static $speed = 10;

public static function getSpeed() {
return self::$speed;
}
}
echo Car::getSpeed();  //调用静态方法
```
静态方法也可以通过变量来进行动态调用
```
$func = 'getSpeed';
$className = 'Car';
echo $className::$func();  //动态调用静态方法
```
静态方法中，$this伪变量不允许使用。可以使用self，parent，static在内部调用静态方法与属性。
```
class Car {
private static $speed = 10;

public static function getSpeed() {
return self::$speed;
}

public static function speedUp() {
return self::$speed+=10;
}
}
class BigCar extends Car {
public static function start() {
parent::speedUp();
}
}
BigCar::start();
echo BigCar::getSpeed();
```
- 对象继承
类的继承用extends关键字
```
<?php
class Car {
public $speed = 0; //汽车的起始速度是0

public function speedUp() {
$this->speed += 10;
return $this->speed;
}
}
//定义继承于Car的Truck类
class Truck extends Car{
//重写speedUp方法
public function speedUp(){
$this->speed += 50;
return $this->speed;
}
}
$car = new Truck();
$car->speedUp();
echo $car->speed;
```
- ####重载
1) php的重载和其他语言中的重载意义不一样，php中的重载指的是动态的创建属性与方法，是通过魔术方法实现的。属性的重载通过__set、__get 、__isset 、__unset来分别实现对不存在属性的赋值、读取、判断属性是否设置、销毁属性。
```
class Car {
private $ary = array();
public function __set($key, $val) {
$this->ary[$key] = $val;
}
public function __get($key) {
if (isset($this->ary[$key])) {
return $this->ary[$key];
}
return null;
}
public function __isset($key) {
if (isset($this->ary[$key])) {
return true;
}
return false;
}
public function __unset($key) {
unset($this->ary[$key]);
}
}
$car = new Car();
$car->name = '汽车';  //name属性动态创建并赋值
echo $car->name;
```
在上面的例子中我们可以看到，本来的`Car`类中是没有`name`属性的，通过属性的重载可以动态的创建`name`属性并进行相应操作。
2) 同理，也可以实现方法的重载，方法的重载是通过__call来实现的，当调用不存在的方法的时候，将会转为参数调用__call方法，当调用不存在的静态方法时会使用__callStatic重载。
```
class Car {
public $speed = 0;

public function __call($name, $args) {
if ($name == 'speedUp') {
$this->speed += 10;
}
}
}
$car = new Car();
$car->speedUp(); //调用不存在的方法会使用重载
echo $car->speed;
```
- 字符串操作
1) 去除字符串的空格
trim去除一个字符串两端空格。
rtrim是去除一个字符串右部空格，其中的r是right的缩写。
ltrim是去除一个字符串左部空格，其中的l是left的缩写。
```
echo trim(" 空格 ")."<br>";
echo rtrim(" 空格 ")."<br>";
echo ltrim(" 空格 ")."<br>";
```
2) 获取字符串长度
```
$str = 'hello';
$len = strlen($str);
echo $len;//输出结果是5
//中文情况
$str = "我爱你";
echo mb_strlen($str,"UTF8");//结果：3，此处的UTF8表示中文编码是UTF8格式，中文一般采用UTF8编码
```
3）字符串截取
```
$str='i love you';
//截取love这几个字母
echo substr($str, 2, 4);//为什么开始位置是2呢，因为substr函数计算字符串位置是从0开始的，也就是0的位置是i,1的位置是空格，l的位置是2。从位置2开始取4个字符，就是love。
$str='我爱你，中国';
//截取中国两个字
echo mb_substr($str, 4, 2, 'utf8');//为什么开始位置是4呢，和上一个例子一样，因为mb_substr函数计算汉字位置是从0开始的，也就是0的位置是我,1的位置是爱，4的位置是中。从位置4开始取2个汉字，就是中国。中文编码一般是utf8格式
```
4) 字符串查找
```
$str = 'I want to study at imooc';
$pos = strpos($str, 'imooc');
echo $pos;//结果显示19，表示从位置0开始，imooc在第19个位置开始出现
```
5) 字符串替换
```
$str = 'I want to learn js';
$replace = str_replace('js', 'php', $str);
echo $replace;//结果显示I want to learn php
```
6) 字符串合并与分割
```
$arr = array('Hello', 'World!');
$result = implode('', $arr);
print_r($result);//结果显示Hello World!
$str = 'apple,banana';
$result = explode(',', $str);
print_r($result);//结果显示array('apple','banana')
```
7) 字符串的反义
反义的目的主要是保证数据传输的安全性
```
$str = "what's your name?";
echo addslashes($str);//输出：what\'s your name?
```