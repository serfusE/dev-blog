# Flask Web Development

## Dependences
 Flask 的主要两大依赖
1. Werkzeug，WSGI 工具库；
2. Jinja2，模版

数据存储方面，用到的是 redis 而不是关系型数据库。redis 是一种内存（in-memory）数据库，仅支持 Unix 系统，但有非官方的 Windows 端。

导入 Flask，同时包含 Jinja2, Werkzeug
```bash
(env) $ pip install flask
```

## Chapter 2. Basic Application Structure
最简单的 Flask 应用：
```python
# hello.py

from flask import Flask
# create Flask instance
app = Flask(__name__) 

# route URL to function
@app.route('/')
def index():
	return '<h1>Hello My App</h1>'

# dynamic route
@app.route('/user/<name>')
def user(name):
	return '<h1>hello, %s!</h1>' % name

# start server
if __name__ == '__main__':
	app.run(debug=True)
```

目录结构：
- Project Directory
	- templates -\> Jinja2 files
	- static -\> images, JS, CSS

### Routes and View Functions
客户端向服务器发送一个请求，Flask app 的实例接受这个请求，对于这个应用实例，它需要知道对于每一条 URL 请求执行什么操作，所以它有一套 URL 对应到 Python 函数的匹配规则。这种对 URL 和函数之间的联系的处理就叫 **route**。

最简单的定义一个 route 的方式是用 `Flask` 实例提供的装饰器 `route` 来修饰函数。如注释 `a`，这样就把 `index()` 函数注册为应用根目录下的 handler。比如域名是 *www.abc.com*，则访问 *http://www.abc.com* 时就会触发这个 `index()` 函数，客户端收到的 response 即是其返回值。

但是这样将响应（HTML 字符串）嵌入到 Python 代码里面非常不利于维护，第三章会介绍更合适的生成响应的方式。

类似 `index()` 这样的函数被称为视图函数（**view function**）。

URL 中尖括号包含的部分称为动态元素（**dynamic component**），视图函数被调用时，动态部分作为参数被传递到 Flask。动态元素默认是字符串类型，类型可以用 `/user/<int:id>` 的形式指定，支持的类型有 `string` `int` `float` 和 `path` 等。`path` 与 `string` 的区别是前者不会把 `/` 作为分隔符。

进入虚拟环境并直接运行 `hello.py`，就可以在 localhost 访问到了。

### 请求、响应周期

#### a. (Application and Request) Contexts
Flask 有两种场景（**context**），四个全局场景变量（**context variables**）：
![](DraggedImage-2.jpeg)

在派发一个请求前，Flask 会激活（也叫入栈）两种场景，当请求处理完毕时移除。通过使用场景，可以在视图或 CLI 追踪到 Flask 应用的各类属性数据，而不必在项目的各个模组里反复导入 `app` 实例。

关于场景的生命周期：当 Flask 应用处理一个请求时，它会入栈一个应用场景和一个请求场景；请求结束时，先出栈请求场景，然后是应用场景。通常来说两者的生命周期都是取决于单个请求的。

在访问场景变量时，如果未先激活场景将会报错
```
>>> from hello import app # Flask instance
>>> from flask import current_app # context variable
>>> current_app.name 
Traceback (most recent call last):
...
RuntimeError: working outside of application context

>>> app_ctx = app.app_context() # obtain application context
>>> app_ctx.push()
>>> current_app.name 
'hello'
>>> app_ctx.pop()
```

#### b. Request Dispatching
用 `Flask` 的 `url_map` 属性可以查看实例中已有的路由表
```
(env) $ python3
>>> from hello import app
>>> app.url_map 
Map([<Rule '/' (HEAD, OPTIONS, GET) -> index>, 
 <Rule '/static/<filename>' (HEAD, OPTIONS, GET) -> static>,
 <Rule '/user/<name>' (HEAD, OPTIONS, GET) -> user>])
```
第二个规则是 Flask 提供的用于访问静态资源的路由。

`(HEAD, OPTIONS, GET)` 代表的是请求方法（**request method**），通过为路由附上请求方法，可以针对同一个 URL 调用不同的视图函数。

#### c. Request Hooks
钩子函数，定义请求派发之前或之后执行的函数，通过装饰器实现。

#### d. Response
这次让视图函数返回一个  `Response` 对象而不是原始字符串
```python
from flask import make_response 

@app.route('/') 
def index():
    response = make_response('<h1>This document carries a cookie!</h1>') 
    response.set_cookie('answer', '42') 
    return response
```

如果要进行重定向，常用 `redirect(another_url_path)`，它也返回一个 `Response` 对象；如果要进行错误处理，`abort(404)` 能返回 404 状态码。不过要注意的是它不会把控制交还给调用它的函数，而是交还网页服务器抛出异常。

```python
from flask import abort 

@app.route('/user/<id>') 
def get_user(id):
    user = load_user(id) 
    if not user:
        abort(404) 
    return '<h1>Hello, %s</h1>' % user.name
```

### Flask 扩展
如果我们要对 Flask 服务器进行配置，可以在 `app.run()` 中添加参数，但更方便的方式是通过命令行传参。

先安装 `flask-script`
```
(venv) $ pip install flask-script
```

用 `Manager` 类接管 Flask app
```python
from flask.ext.script import Manager 
manager = Manager(app)

# ...

if __name__ == '__main__':
    manager.run()
```

这时候再运行程序，就会提示提供参数了
```
(env) $ python3 hello.py
usage: hello.py [-?] {shell,runserver} ...

positional arguments:
  {shell,runserver}
	shell            Runs a Python shell inside Flask application context.
	runserver        Runs the Flask development server i.e. app.run()

optional arguments:
  -?, --help         show this help message and exit
```
设置 host（a.b.c.d:5000）示例
```
(env) $ python hello.py runserver --host 0.0.0.0 
```

## Chapter 3. Templates
从第二章了解到，视图函数的任务是针对请求生成响应，但如果请求比较复杂，比如用户发起一个注册请求，视图函数需要 1) 访问数据库创建新记录 2) 生成响应发送给客户端。这样就把业务逻辑（**business logic**）和展示逻辑（**presentation logic**）混淆在了一起，导致代码难以理解且不利于维护。

把展示层逻辑转移到模版（**template**）即可以提高可维护性。模版是包含响应文本的 HTML 文件，数据由占位符变量表示。而把变量替换为真实的数据的过程就叫做渲染（**rendering**）。

### Jinja2 模版渲染引擎
Jinja2 模版语法和变量的使用

```python
from flask import Flask, render_template # import rendering method
app = Flask(__name__)

@app.route('/')
def index():
	return render_template('index.html') # refer to templates/index.html

@app.route('/user/<name>')
def user(name):
	return render_template('user.html', name=name) # dynamic component
```

在模版文件中使用构造器 `{{ variable }}` 来获得变量：
```html
<!-- templates/user.html -->
<h1>Hello, {{ name }}!</h1>
```

#### Jinja2 variable filters
Jinja2 提供了一些实用的变量过滤器，查阅[官方文档](http://jinja.pocoo.org/docs/2.10/templates/)获取更多用法。
![](DraggedImage-3.jpeg)
例如，Jinja2 默认会对变量进行转义，如果需要传递 HTML 代码，用 `safe` 来避免转义：`{{ variable_name|safe }}`。但要注意，永远不要对不可信变量使用 `safe` 过滤（比如用户提交的表单）。

#### Control Structures
条件判断：
```
{% if user %} 
    Hello, {{ user }}! 
{% else %}
    Hello, Stranger! 
{% endif %}
```

循环语句：
```
{% for comment in comments %}
    {{ comment }}
{% endfor %}
```

宏（macro），类似 Python 中的函数：
```html
{% macro render_comment(comment) %} 
    <li>{{ comment }}</li> 
{% endmacro %}

<ul>
    {% for comment in comments %} 
        {{ render_comment(comment) }} 
    {% endfor %} 
</ul>
```

为了提高 macros 的可用性，可以放到一个单独的 html 文件中，在模版通过 `import` 导入：
```
{% import 'macros.html' as macros %} 
{{ macros.render_comment(comment) }} 
```

重复使用的模版代码也可以单独拿出来，在需要的时候导入：
```
{% include 'common.html' %}
```

更进一步，可以用模版继承来完善代码结构。首先创建一个基本模版：
```html
<html> 
<head>
    {% block head %}
    <title>{% block title %}{% endblock %} - My Application</title>
    {% endblock %} 
</head> 
<body>
    {% block body %}
    {% endblock %} 
</body> 
</html>
```
实现继承的模版：
```html
{% extends "base.html" %} 
{% block title %}Index{% endblock %} 
{% block head %}
    <!-- 如果被继承的 block 中包换元素，用 super() 获取 -->
    {{ super() }} 
    <style>
    </style> 
{% endblock %} 
{% block body %} 
<h1>Hello, World!</h1> 
{% endblock %}
```

**由于修饰笔记耗费太多的时间，为了加快进度，从这里开始往后改变以笔记为首的学习方式，多做引用和直接在原书上批注（这样的话可能以后随时按需拓展）**

### 与 Bootstrap 的整合
Bootstrap 是 Twitter 的开源框架，它提供创建网页的用户界面接口，兼容性好。简单的说就是一个 CSS 和 JS 库。

使用 Bootstrap 有多种方式：1）将编译过的 CSS 和 JS 文件下载下来放到项目中；2）用包管理器安装 [Flask-Bootstrap 扩展](https://pythonhosted.org/Flask-Bootstrap/basic-usage.html)；3）用 Bootstrap 官方提供的免费 CDN（通过 HTML `<link>`）。在这里，我们使用第二种方式。

如何加载 Bootstrap：
1. 通过虚拟环境安装包
```
(env) $ pip install flask-bootstrap
```
2. 在 Python 程序中导入 `Bootstrap` 模块
3. 用 Flask 应用作为构造器参数实例化 `Bootstrap`
```python
from flask_bootstrap import Bootstrap
app = Flask(__name__)
bootstrap = Bootstrap(app)
```
4. 通过 Jinja2 导入
```Jinja2
{% extends "bootstrap/base.html" %}
```

Bootstrap 本身是在 base 模版的 `styles` 和 `scripts` 块里面定义的，所以如果要继承这两个块，一定要加上 `{{ super() }}`。

tip：Flask-Bootstrap 有一个更新、更轻量的版本 [Bootstrap-Flask](https://bootstrap-flask.readthedocs.io/en/latest/)。

### 链接处理
Flask 提供了一个 `url_for()` 方法。几种用法举例：
1. 为 URL 填入动态元素
`url_for('user', name='john')` -\> */user/john.*
2. 设置请求参数字符串
`url_for('index', page=2)` -\> */?page=2*
3. 指定为外部链接
`url_for('index', _external=True)` -\> *http://localhost:5000*
4. 链接到 static 目录下的 CSS 文件
`url_for('static', filename='css/styles.css')` -\> */static/css/styles.css*

下面是用 Jinja2 导入一些头部文件的示例：
```Jinja2
{% block head %} 
{{ super() }} <!—- 注意要继承 Bootstrap 中的 head -—>
<link rel="shortcut icon" href="{{ url_for('static', filename = 'favicon.ico') }}" type="image/x-icon"> 
<link rel="icon" href="{{ url_for('static', filename = 'favicon.ico') }}" type="image/x-icon"> 
{% endblock %}
```

### 用 Flask-Moment 实现时区的本地化
为了统一标准，服务器时间都使用 UTC 标准时间，时区现实的本地化在客户端浏览器完成。moment.js 是一个开源的客户端的 JS 库，Flask 的拓展将它和 Jinja2 模版进行了整合，这就是 Flask-Moment。

安装：
```
(env) $ pip install flask-moment
```
初始化：
```python
from flask.ext.moment import Moment 
moment = Moment(app)
```

除了 moment.js，Flask-Moment 还依赖于 jquery.js，不过 Bootstrap 已经包含了后者，所以我们只需要手动导入 moment.js。

1. Import module
```python
from flask_moment import 
```
2. Initialization
```python
moment = Moment(app) 
# `app` refers to Flask instance
# again, the name in the left is irrelevant.
```
3. Pass time object to template
```python
@app.route('/')
def index():
	return render_template('index.html', current_time=datetime.utcnow())
```
4. Present local date and time in Jinja2
```Jinja2
<p>The local date and time is {{ moment(current_time).format('LLL') }}.</p>
<p>That was {{ moment(current_time).fromNow(refresh=True) }}.</p>
{# 
The local date and time is September 17, 2018 8:29 PM.
That was a few seconds ago. 
*}
```

## Chapter 4. Web Forms
这一章讨论模版与用户的交互。

通过 POST 提交的表单用 `request.form` 获取

**WTForms** 是 Python web 开发中一个灵活的表单验证即渲染库。它与框架无关，可以搭配任何一种 web 框架和模版引擎使用。

安装：
```
(env) $ pip install flask-wtf
```

对于 Flask 的一些功能，我们有必要为应用配置一个密钥
```python
app = Flask(__name__)
app.config['SECRET_KEY'] = 'hard to guess string'
```
需要密钥的功能包括 session 的加密签名、CSRF 保护等，而后者又是 WTFroms 的必须项，所以，如果不配置密钥就使用 WTForms，将会出现运行时错误 **A secret key is required to use CSRF**。

### CSRF 保护
跨站请求伪造（cross-site request forgery, CSRF）是一种恶意的攻击，盗用经过验证的用户身份执行未经授权的命令，如发送邮件、发消息、盗取账号、转账等隐私风险。

什么情况下会发生这种情况呢？比如你在一家安全措施薄弱的银行网站进行了登录操作，然后打开了另一个不安全网站，这个网站就可以利用你的 cookie 中的登录信息，伪造一个由浏览器发出的转账请求。所以 CSRF 利用到的漏洞在于 web 的隐式身份验证机制，WEB 的身份验证机制虽然可以保证一个请求是来自于某个用户的浏览器，但却无法保证该请求是用户批准发送的[^1]。

要实现 CSRF 保护，Flask-WTF 需要应用配置一个密匙。

### 表单类
当使用 Flask-WTF 时，每一个表单都由 `Form` 的一个子类表示，这个类定义了表单中的元素，每一个都用一个对象表示。可以为每个单独的对象附上一或多个校验器（validator）来确保用户提交内容有效。

像这样定义一个表单类：
```python
from flask_wtf import FlaskForm
from wtforms import StringField, SubmitField 
from wtforms.validators import Required

class NameForm(FlaskForm):
    name = StringField('What is your name?', validators=[Required()]) 
    submit = SubmitField('Submit')
```

表单中用到的各类元素都是 WTFroms 下的，最后由 Flask-From （ WTFroms `Form` 的子类）封装。

WTForms 支持的 HTML 域：
![](DraggedImage.jpeg)
校验器：
![](DraggedImage-1.jpeg)

[^1]:	[http://www.cnblogs.com/hyddd/archive/2009/04/09/1432744.html](http://www.cnblogs.com/hyddd/archive/2009/04/09/1432744.html)