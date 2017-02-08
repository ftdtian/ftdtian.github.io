---
title: IOC+DI
date: 2016-11-27 02:33:45
tags: 业务驱使技术
category: "业务驱使技术"
---

### 什么是IoC
IoC(控制反转)只是一种设计思想，它告诉你应该如何做，来解除相互依赖模块的耦合，它为相互依赖的组件提供抽象，将依赖（低层模块）对象的获得交给第三方（系统）来控制，即依赖对象不在被依赖模块的类中直接通过new来获取。

讲到IoC就得说一下DI，IoC是一种设计思想，DI就是基于IoC思想的具体实现方式。
DI，依赖注入，从字面意思就可以看出，依赖是通过外接注入的方式来实现的。这就实现了解耦，而DI的方式通常有两种：
- 构造器注入
- setter注入


### IOC+DI举例

先看一个简单的例子，有一个Order类，订单需要入库，我们需要一个数据库实例，我们使用Mysql。
```php
class Order {
	
    private $db = new Mysql();

    public function add()
    {
        $db->add();
    }
}

class Mysql {

    public function add()
    {
        echo '新的订单入库';
    }
}
```
可以看出Order类依赖于Mysql类，如果此时数据库实例需要修改成Oracle，我们的Order类就需要修改
```php 
class Order {
	
    private $db = new Oracle();

    public function add()
    {
        $db->add();
    }
}

```
所有我们需要的DB实例，由我自己主动new，内部的依赖性使得代码耦合性较高。我们需要解除这种依赖关系，
Order需要的DB实例由外部进行提供。
```php
interface DB {
    public function add();
}

class Mysql implements DB {
    public funciton add() 
    {
        //to do
    }
}

class Oracle implements DB {

    public funciton add() 
    {
    	//to do
    }
}



class Order {
	
    private $db;

    public function __construct(DB $db)
    {
    	$this->db = $db;
    }
    
    public function add()
    {
    	$this->db->add();
    }
    
}

$mysql = new Mysql();
$order = new Order($mysql);

```

这里代码使用了构造函数注入，Order类内部不再依赖于DB实例，而是由程序外部提供，你需要什么实例，就给你什么实例。原先的依赖关系就不存在了，假设所有的DB类是一个容器，Order类依赖于此容器。

>控制反转即IoC (Inversion of Control)，它把传统上由程序代码直接操控的对象的调用权交给容器，通过容器来实现对象组件的装配和管理。所谓的“控制反转”概念就是对组件对象控制权的转移，从程序代码本身转移到了外部容器。


### 设计一个IoC容器
我们已经知道如何解决依赖关系了，此时我们需要设计一个良好的IoC容器，来健壮我们的代码，这里分享一个同事写的DI容器类
```php
class DI implements \ArrayAccess
{
    private static $_bindings = array();//服务列表
    public static $_instances = array();//已经实例化的服务

    /**
     * 获取服务
     *
     * @param $name
     * @return mixed|null|object
     */
    public static function get($name)
    {
        //先从已经实例化的列表中查找
        if(isset(self::$_instances[$name])){
            return self::$_instances[$name];
        }

        //检测有没有注册该服务
        if( ! isset(self::$_bindings[$name])){
            $msg = '<<'.$name.'>> must be set';
            trigger_error($msg);
            return null;
        }
        $concrete = self::$_bindings[$name]['class'];//对象具体注册内容
        $params = self::$_bindings[$name]['params'];//配置
        $obj = null;
        //匿名函数方式
        if($concrete instanceof \Closure){
            $obj = call_user_func_array($concrete,$params);
        }elseif(is_string($concrete)){//字符串方式
            if(empty($params)){
                $obj = new $concrete;
            }else{
                //带参数的类实例化，使用反射
                $class = new \ReflectionClass($concrete);
                $obj = $class->newInstanceArgs($params);
            }
        }
        //如果是共享服务，则写入_instances列表，下次直接取回
        if(self::$_bindings[$name]['shared'] == true && $obj){
            self::$_instances[$name] = $obj;
        }

        return $obj;
    }

    /**
     * 检测是否已经绑定
     *
     * @param $name
     * @return bool
     */
    public static function has($name)
    {
        return isset(self::$_bindings[$name]) or isset(self::$_instances[$name]);
    }

    /**
     * 卸载服务
     *
     * @param $name
     * @return bool
     */
    public static function remove($name)
    {
        unset(self::$_bindings[$name],self::$_instances[$name]);
        return true;
    }

    /**
     * 设置服务
     *
     * @param $name
     * @param $class
     * @param $params
     * @return bool
     */
    public static function set($name, $class, $params = array())
    {
        self::_registerService($name, $class, $params);
        return true;
    }

    /**
     * 设置共享服务
     *
     * @param $name
     * @param $class
     * @param $params
     */
    public static function setShared($name, $class, $params = array())
    {
        self::_registerService($name, $class, $params, true);
    }

    /**
     * 注册服务
     *
     * @param $name
     * @param $class
     * @param $params
     * @param bool|false $shared
     */
    private static function _registerService($name, $class, $params = array(), $shared = false)
    {
        self::remove($name);
        if( ! ($class instanceof \Closure) && is_object($class)){
            self::$_instances[$name] = $class;
        }else{
            self::$_bindings[$name] = array('class'=>$class,'shared'=>$shared,'params'=>$params);
        }
    }

    /**
     * ArrayAccess接口,检测服务是否存在
     *
     * @param mixed $offset
     * @return bool
     */
    public function offsetExists($offset)
    {
        return self::has($offset);
    }

    /**
     * ArrayAccess接口,以$di[$name]方式获取服务
     *
     * @param mixed $offset
     * @return mixed|null|object
     */
    public function offsetGet($offset)
    {
        return self::get($offset);
    }

    /**
     * ArrayAccess接口,以$di[$name]=$value方式注册服务，非共享
     *
     * @param mixed $offset
     * @param mixed $value
     * @return bool
     */
    public function offsetSet($offset, $value)
    {
        return self::set($offset,$value);
    }

    /**
     * ArrayAccess接口,以unset($di[$name])方式卸载服务
     *
     * @param mixed $offset
     * @return bool
     */
    public function offsetUnset($offset)
    {
        return self::remove($offset);
    }
}

```
简单介绍一下此类：
>1.静态属性$_bindings存放已注册的服务(并不一定实例化)，$_instances存放了已经实例化的服务
2.匿名函数类Closure搭配call_user_func_array
3.带参数的类实例化，使用反射ReflectionClass
4.实例化类是否共享(可选择是否将实例化的对象存放到$_instances中)
5.实现了ArrayAccess接口，可以以数组形式注册、获取服务

使用方法：
在框架的入口，将所需要的服务类注册到此容器中，比如Nosql、DB、Log、Http、MQ等。
此处我们已redis操作类为例
```php
public function _initDI()
{
		
        $configArr = \Yaf\Application::app()->getConfig();
        
        //直接实例化注册(不建议)
        $redis = new Redis($config['redis']['test']);
        DI::setShared('redis', $redis);
        //或
        $di = new DI();
        //以数组方式注入服务，实例化非共享
        $di['redis'] = $redis;
        
        /*------------------------------------------------*/
        
        //延迟实例化注册
        DI::setShard('redis', function() use(config) {
        	return new Redis($config['redis']['test']);
        });
        //或
        DI::setShard('redis', 'Redis', $config['redis']['test']);
        //或
        $di = new DI();
        $di['redis'] = function() use ($config) {
        	return new Redis($config['redis']['test']);
        };
        
      
}
```

我们已经在容器中注册了我们需要的服务类（并没有实例化，只有在我们需要的时候才进行实例化）
```php
public function test()
{	
    //获取Redis对象
    $redisObj = DI::get('redis');
    //或者
    $di = new DI();
    if (isset($di['redis'])) {
    	$redisObj = $di['redis'];
    }
    
}
```

### 总结
IoC的核心思想，我们所依赖的服务类不由自己管理，将其统一交给不使用它的第三方容器管理。
1.服务或者组件集中管理，可配置、易管理。
2.减少类之间的依赖关系，也就是说降低代码的耦合度。
3.不要在父类的构造方法各种实例化了其他类，然后子类来一发parent::__construct 了。。。






