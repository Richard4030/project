Django 温故
===========

> **Auth** : 张旭<br>
> **Date** : 2017-12-16<br>
> **Email**: <zhangxu@1000phone.com>


#### [HTTP Objects](https://docs.djangoproject.com/en/2.0/ref/request-response/)

- HttpRequest
    - 自身属性
        * request.path -> `/foo/bar/`
        * request.method -> `POST`, `GET`
        * 类字典属性
            * request.GET
            * request.POST
            * request.COOKIES
            * request.FILES -> `{name1: file1, name2: file2, ...}`
            * request.META['REMOTE_ADDR']
            * request.META['HTTP_USER_AGENT']
    - 中间件添加的属性
        * request.session
        * request.user
    - 方法
        * request.get_full_path() -> `/foo/bar/?a=123`
        * request.get_signed_cookie(key)
- HttpResponse
    - 属性
        * response.status_code
        * response.content
    - 方法
        * response.set_cookie(key, value, max_age=None)
- JsonResponse
    - `response = JsonResponse({'a': 12, 'b': 'xyz'})`

#### Django 中间件
- 面向切片编程
- 最简单的中间件

    ```python
    def simple_middleware(get_response):
        # do_something_for_init()

        def middleware(request):
            # do_something_before_views(request)

            response = get_response(request)  # views 函数在这里执行

            # do_something_after_views(response)

            return response

        return middleware
    ```

- 中间件类

    ```python
    class MyMiddleware:
        def __init__(self, get_response):
            self.get_response = get_response
    
        def __call__(self, request):
            # do_something_before_views(request)
            response = self.get_response(request)
            # do_something_after_views(response)
            return response
    
        def process_view(self, request, view_func, view_args, view_kwargs):
            response = view_func(*view_args, **view_kwargs)
            return response
    ```

- Django-1.10 以前的中间件

    ```python
    from django.utils.deprecation import MiddlewareMixin

    class MyMiddleware(MiddlewareMixin):
        def process_request(self, request):
            pass

        def process_view(self, request, view_func, view_args, view_kwargs):
            pass

        def process_response(self, request, response):
            return response
    ```

- 执行顺序
    - process_request, process_view 从上往下执行
    - process_response 从下往上执行
- [内置中间件的排序](https://docs.djangoproject.com/en/2.0/ref/middleware/#middleware-ordering)

#### Form 表单
- 核心功能：数据验证

- form 的 method 只能是 POST 或 GET

- method=GET 时, 表单提交的参数会出现在 URL 里

- 属性和方法
    - `form.is_valid()`
    - `form.has_changed()`
    - `form.clean_<field>()`
    - `form.cleaned_data['fieldname']`

- Form 的定义和使用

    ```python
    from django.forms import Form
    from django.forms import IntegerField, CharField, DateField, ChoiceField

    class TestForm(Form):
        TAGS = (
            ('py', 'python'),
            ('ln', 'linux'),
            ('dj', 'django'),
        )
        fid = IntegerField()
        name = CharField(max_length=10)
        tag = ChoiceField(choices=TAGS)
        date = DateField()

    data = {'fid': 'abc123', 'name': '1234567890', 'tag': 'dj', 'date': '2017-12-17'}
    form = TestForm(data)
    print(form.is_valid())
    print(form.cleaned_data)  # cleaned_data 属性是 is_valid 函数执行时动态添加的
    print(form.errors)
    ```


- ModelForm

    ```python
    class UserForm(ModelForm):
        class Meta:
            model = User
            fields = ['name', 'birth']
    ```

#### 模板
- base.html 模板推荐布局

    ![layout](./img/layout.jpg)

    ```html
    <!DOCTYPE html>
    <html>
        <head>
            <title>{{title}}</title>
            <link rel="stylesheet" type="text/css" href="/static/css/style.css">
            {% block "ext_css" %}{% endblock %}
        </head>
        <body>
            <!-- {% block "navbar" %}{% endblock %} -->
            {% block "sidebar" %}{% endblock %}
            {% block "content" %}{% endblock %}
            <!-- {% block "foot" %}{% endblock %} -->
            {% block "ext_js" %}{% endblock %}
        </body>
    </html>
    ```

- [内建 Tags](https://docs.djangoproject.com/en/2.0/ref/templates/builtins/#ref-templates-builtins-tags)

    - `csrf_token`

        ```html
        <form>
        	{% csrf_token %}
            <input type="text" name="x" value="123">
        </form>
        ```

    - `for...endfor` 中的变量
        * `forloop.counter`     从 1 开始计数
        * `forloop.counter0`    从 0 开始计数
        * `forloop.revcounter`  逆序计数到 1
        * `forloop.revcounter0` 逆序计数到 0
        * `forloop.first`       是否是循环中的第一个
        * `forloop.last`        是否是循环中的最后一个
        * `forloop.parentloop`  用于引用上级循环中的变量, 如 `{{ forloop.parentloop.counter }}`
    - empty 子句
        ```html
        {% for x in lst %}
            <div>...</div>
        {% empty %}
            <div>Sorry</div>
        {% endfor %}
        ```

    - load: 加载自定义 Tag `{% load foo.bar %}`
    - url: 根据 url name 替换 `{% url 'your-url-name' v1 v2 %}`
    - static
        ```html
        {% load static %}
        <img src="{% static "img/smile.jpg" %}">
        ```

        或
        ```html
        {% load static %}
        <img src="{% get_static_prefix %}img/smile.jpg">
        ```

- [内建的 filter](https://docs.djangoproject.com/en/2.0/ref/templates/builtins/#built-in-filter-reference)
    - safe 和 escape: `{{ var|safe|escape }}`

- [使用 Jinja2 替换内置模板引擎](https://docs.djangoproject.com/en/2.0/topics/templates/#django.template.backends.jinja2.Jinja2)

#### ORM
- 什么是 ORM
- CURD (Create/Update/Retrieve/Delete)
- [Field](https://docs.djangoproject.com/en/2.0/ref/models/fields/)
- Field 选项
    * `null`    针对数据库, 允许数据库该字段为 Null
    * `blank`   针对 Model 本身, 允许传入字段为空. blank 为 True 时, 对数据库来说, 该字段依然为必填项
    * `default` 尽量使用 default, 少用 null 和 blank
    * `choices`
    * `primary_key` 非必要时不要设置, 用默认 id, 保持条目自增、有序
    * `unique`
    * `db_index`    (True | False)
    * `max_length`
    * `auto_now`     每次 save 时，更新为当前时间
    * `auto_now_add` 只记录创建时的时间, 保存时不更新
- [QuerySet](https://docs.djangoproject.com/en/2.0/ref/models/querysets/)
    - 方法
        * 创建: `create() / get_or_create() / update_or_create() / bulk_create()`
        * 条件过滤和排除: `filter() / exclude()`
        * 只加载需要的字段: `only() / defer()`
        * `order_by() / count() / exists()`
        * `latest() / earliest()`
        * `first() / last()`
    - 查找条件
        * `filter(id__in=[123, 555, 231])`
        * `filter(id__range=[123, 456])`
        * `filter(name__contains='123')`
        * `filter(name__regex='^\w+\d+')`
        * `filter(id__lte=123)`
        * `gt / gte / lt / lte`
- 其他 ORM
    * sqlalchemy
    * [peewee](http://docs.peewee-orm.com/en/latest/)
- 主键和外键约束
    - 内部系统、传统企业级应用可以使用 (需要数据量可控，数据库服务器数量可控)
    - 互联网行业不建议使用
        - 性能缺陷
        - 不能用于分布式环境
        - 不容易做到数据解耦

#### Cache
- 默认缓存: `from django.core.cache import cache`
- BACKEND: `DatabaseCache / MemcachedCache / LocMemCache`
- LOCATION: IP:Port 绑定, 只有一个时配制成字符串链接, 有多台时配制为列表
- 使用 Redis 做缓存

    ```python
    CACHES = {
        "default": {
            "BACKEND": "django_redis.cache.RedisCache",
            "LOCATION": "redis://127.0.0.1:6379/1",
            "OPTIONS": {
                "CLIENT_CLASS": "django_redis.client.DefaultClient",
                "PICKLE_VERSION": -1,
            }
        }
    }
    ```

- 基本方法
    * `cache.set(key, value, timeout=None)`
    * `cache.get(key, default=None)`
    * `cache.delete(key)`
    * `cache.incr('num')`
    * `cache.decr('num')`
    * `cache.get_or_set(key, default, timeout=None)`
    * `cache.set_many({'a': 1, 'b': 2, 'c': 3})`
    * `cache.get_many(['a', 'b', 'c'])`
- 全站缓存中间件: `django.middleware.cache.UpdateCacheMiddleware`
    - 前置中间件
    - 缓存期限: CACHE_MIDDLEWARE_SECONDS
- 页面缓存装饰器: `from django.views.decorators.cache import cache_page`
- 属性缓存装饰器: `from django.utils.functional import cached_property`
- pickle
    * `data = dumps(obj, -1)`
    * `obj = loads(data)`

#### Cookie 和 Session
- 产生过程
    1. *浏览器* 发起请求访问服务器
    2. *服务器* 为 *浏览器* 创建 Session 对象, 其中包含一个全局唯一的 ID —— `session_id`
    3. *服务器* 将 session_id 写入 response 的 Cookie 中, 传回 *浏览器*
    4. *浏览器* 取出 session_id, 并将其存入 Cookie
    5. 后续每次请求 session_id 会随 Cookie 一同传到 *服务器*
    6. *服务器* 取出 session_id 并找出用户之前存储的状态信息
- Cookie: `response.set_cookie(key, value, max_age=None)`
- Session 配置
    1. 开启 Session 中间件: `django.contrib.sessions.middleware.SessionMiddleware`
    2. 配置缓存
    3. 配置 Session 引擎: `SESSION_ENGINE = "django.contrib.sessions.backends.cache"`
- 可选项
    * `SESSION_COOKIE_AGE`              缓存时间, 默认 2 周
    * `SESSION_COOKIE_NAME`             Session 名, 默认 'sessionid'
    * `SESSION_EXPIRE_AT_BROWSER_CLOSE` 浏览器关闭页面时, Session 是否设为过期
    * `SESSION_SAVE_EVERY_REQUEST`      每次请求时, 是否强制保存一次 Session
- 用法
    * `request.session.session_key`     查看 session_id
    * `request.session.modified`        session 是否发生过修改
    * `request.session['uid'] = 1234`   当 session 发生更改时会自动保存
    * `request.session.get('uid')`      取值
    * `request.session.save()`          手动保存

#### Logging
- 日志级别
    * DEBUG
    * INFO
    * WARNING
    * ERROR
    * FATAL
- 使用
    * logger.debug('xxxxxxxx')
    * logger.info('xxxxxxxx')
    * logger.warning('xxxxxxxx')
    * logger.error('xxxxxxxx')
    * logger.fatal('xxxxxxxx')
- 查找、分析
    * tail
    * head
    * less
    * awk
    * grep
- [配置](https://docs.python.org/2/library/logging.html)

    ```python
    LOGGING = {
        'version': 1,
        'disable_existing_loggers': True,
        'formatters': {
            'simple': {
                'format': '%(asctime)s %(module)s.%(funcName)s: %(message)s',
                'datefmt': '%Y-%m-%d %H:%M:%S',
            },
            'verbose': {
                'format': '%(asctime)s %(levelname)s [%(process)d-%(threadName)s] '
                          '%(module)s.%(funcName)s line %(lineno)d: %(message)s',
                'datefmt': '%Y-%m-%d %H:%M:%S',
            }
        },

        'handlers': {
            'inf': {
                'class': 'logging.handlers.TimedRotatingFileHandler',
                'filename': 'info.log',
                'when': 'W3',  # 每周一切割日志
                'backupCount': 5,
                'formatter': 'simple',
                'level': 'INFO',
            },
            'err': {
                'class': 'logging.handlers.TimedRotatingFileHandler',
                'filename': 'error.log',
                'when': 'D',  # 每天切割日志
                'backupCount': 30,
                'formatter': 'verbose',
                'level': 'WARNING',
            }
        },

        'loggers': {
            'inf': {
                'handlers': ['inf'],
                'level': 'DEBUG',
                'propagate': True,
            },
            'err': {
                'handlers': ['err'],
                'level': 'DEBUG',
                'propagate': True,
            }
        }
    }
    ```

#### Django 的性能
- Django 自身优化
    - 充分之用缓存
    - 惰性求值和迭代器
    - 尽量使用 `defer()` 和 `only()` 查找
    - 尽量使用 `count()` 和 `exists()`
    - 模板中 `{% block %}` 性能优于 `{% include %}`
    - [开启模板缓存](https://docs.djangoproject.com/en/2.0/ref/templates/api/#django.template.loaders.cached.Loader)
    - **不要使用外键！不要使用外键！不要使用外键！**
- 其他优化
    - I/O 密集型: 异步化
        - 请求异步化
        - 数据操作异步化
        - gevent, asyncio, aiopg, aiohttp, tornado
    - 计算密集型
        - 耗时操作用 [Celery](http://docs.jinkan.org/docs/celery/) 等工具异步完成
    - 分库分表
        - 取余、哈希
        - 范围
        - 一致性哈希
    - 索引优化
    - 慢查询优化
    - Gunicorn 开启多进程模式利用多核
    - PyPy
    - Cython
