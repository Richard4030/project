#day15 Django用户认证系统
###1. 拓展User模型
	Django 用户认证系统提供了一个内置的 User 对象，用于记录用户的用户名，密码等个人信息。
	对于 Django 内置的 User 模型， 仅包含以下一些主要的属性：
		username，即用户名
		password，密码
		email，邮箱
		first_name，名
		last_name，姓
	对于一些网站来说，用户可能还包含有昵称、头像、个性签名等等其它属性。
	因此仅仅使用 Django 内置的 User 模型是不够。
	好在 Django 用户系统遵循可拓展的设计原则。
	我们可以方便地拓展 User 模型。
####1.1 继承 AbstractUser 拓展用户模型
	这是推荐做法。
	事实上，查看 User 模型的源码就知道，User 也是继承自 AbstractUser 抽象基类。
	而且仅仅就是继承了 AbstractUser，没有对 AbstractUser 做任何的拓展。
	所以，如果我们继承 AbstractUser，将获得 User 的全部特性，而且还可以根据自己的需求进行拓展。
	打开 users/models.py 文件，写上我们自定义的用户模型代码：
	user/models.py
		from django.db import models
		from django.contrib.auth.models import AbstractUser
		# Create your models here.


```python
	class User(AbstractUser):
	    nickname = models.CharField(max_length=50,blank=True)
	    headshot = models.ImageField(upload_to='avatar/%Y/%m/%d',default='default.jpg')
	    signature = models.CharField(max_length=128, default='This guy is too lazy to leave anything here!')
	
	    class Meta(AbstractUser.Meta):
	        pass
为了让 Django 用户认证系统使用我们自定义的用户模型,必须要在settings.py里通过 AUTH_USER_MODEL 指定自定义用户模型所
在的位置，即需要如下设置：
	django_auth_example/settings.py
	# 其它设置...
	AUTH_USER_MODEL = 'users.User'	
即告诉 Django，使用 users 应用下的 User 用户模型。
然后在blog/models.py中修改Post的作者外键为新的User类：
即把	from django.contrib.auth.models import User删除。
加上：from users.models import User
然后执行数据库迁移命令。
如报下面这种错误：
django.db.migrations.exceptions.InconsistentMigrationHistory: Migration admin.0001_initial is applied before its dependency users.0001_initial on database 'default'.
删除数据库中除了auth_user的所有表重新执行数据库迁移命令。
```
###2. 注册
	用户注册就是创建用户对象，将用户的个人信息保存到数据库里。
####2.1 编写用户注册表单
	Django 用户系统内置了登录、修改密码、找回密码等视图
	但是唯独用户注册的视图函数没有提供，这一部分需要我们自己来写。
	Django 已经内置了一个用户注册表单：django.contrib.auth.forms.UserCreationForm，
	不过这个表单的一个小问题是它关联的是 django 内置
	的 User 模型，从它的源码中可以看出：	
		class UserCreationForm(forms.ModelForm):
		    ...
		    class Meta:
		        model = User
		        fields = ("username",)
		        field_classes = {'username': UsernameField}
	我们可以继承它，对它做一点小小的修改就可以了。
	在 users 应用下新建一个 forms.py 文件用于存放表单代码，然后写上如下代码：
		from django.contrib.auth.forms import UserCreationForm
		from .models import User
		
		class RegisterForm(UserCreationForm):
		    class Meta(UserCreationForm.Meta):
		        model = User
		        fileds = ('username', 'email', 'nickname', 'headshot', 'signature')
####2.2 编写用户注册视图函数
```python
users/views.py
	from django.shortcuts import render, redirect
	from .forms import RegisterForm
	from django.urls import reverse
	# Create your views here.
	
	def register(request):
	    if request.method == 'POST':
	        form = RegisterForm(request.POST, request.FILES)
			
	        if form.is_valid():
	            form.save()
	            return redirect(reverse('blog:index'))
	    else:
	        form = RegisterForm()
	    retun render(request, 'users/register.html', {'form':form})
```
####2.3 设置 URL 模式
	users/urls.py
		from django.conf.urls import url
		from . import views
		
		app_name = 'users'
		urlpatterns = [
		    url(r'^register/', views.register, name='register'),
		]
	接下来需要在工程的 urls.py 文件里包含 users 应用的 URL 模式
	django_auth_example/urls.py
	
		from django.conf.urls import url, include
		from django.contrib import admin
		
		urlpatterns = [
		    url(r'^admin/', admin.site.urls),
			...
		    url(r'^users/', include('users.urls')),
		]
####2.4 编写注册页面模板
	已提供模板。在模板中写入下面代码：
		<form class="form" action="{% url 'users:register' %}" method="post">
	        {% csrf_token %}
	        {% for field in form %}
	            {{ field.label_tag }}
	            {{ field }}
	            {{ field.errors }}
	            {% if field.help_text %}
	                <p class="help text-small text-muted">{{ field.help_text|safe }}</p>
	            {% endif %}
	        {% endfor %}
###3 登录
	Django 已经为我们写好了登录功能的全部代码。
	我们不必像之前处理注册流程那样费劲了。
	只需几分钟的简单配置，就可为用户提供登录功能。
####3.1 引入内置的 URL 模型
	Django 内置的登录、修改密码、找回密码等视图函数对应
	的 URL 模式位于django.contrib.auth.urls.py 中。
	首先在工程的 urls.py 文件里包含这些 URL 模式。
		url(r'^users/', include('django.contrib.auth.urls')),
	这将包含以下的 URL 模式：
		^users/login/$ [name='login']
		^users/logout/$ [name='logout']
		^users/password_change/$ [  name='password_change']
		^users/password_change/done/$ [name='password_change_done']
		^users/password_reset/$ [name='password_reset']
		^users/password_reset/done/$ [name='password_reset_done']
		^users/reset/(?P<uidb64>[0-9A-Za-z_\-]+)/(?P<token>[0-9A-Za-z]{1,13}-[0-9A-Za-z]{1,20})/$ [name='password_reset_confirm']
		^users/reset/done/$ [name='password_reset_complete']
####3.2 设置模板路径
	默认的登录视图函数渲染的是 registration/login.html 模板。
	因此需要在 templates/ 目录下新建一个 registration 文件夹。
	再在 registration/ 目录下新建 login.html 模板文件。
####3.3 编写登录模板
	登录模板的代码和注册模板的代码十分类似。
	已提供模板。
	关注表单部分的代码：
		<form class="form" action="{% url 'login' %}" method="post">
		  {% csrf_token %}
		  {{ form.non_field_errors }}
		  {% for field in form %}
		    {{ field.label_tag }}
		    {{ field }}
		    {{ field.errors }}
		    {% if field.help_text %}
		        <p class="help text-small text-muted">{{ field.help_text|safe }}</p>
		    {% endif %}
		  {% endfor %}
		  <button type="submit" class="btn btn-primary btn-block">登录</button>
		</form>
	需要特别说明的是{{ form.non_field_errors }}。
	这显示的同样是表单错误，但是显示的表单错误是和具体的某个表单字段无关的。
	相对 {{ field.errors }}显示的是具体某个字段的错误。
	比如对于字段 username，如果用户输入的 username 不
	符合要求，比如太长了或者太短了，表单会在 username 
	下方渲染这个错误。
	但有些表单错误不和任何具体的字段相关，比如用户输入的
	用户名和密码无法通过验证，这可能是用户输入的用户名不
	存在，也可能是用户输入的密码错误，因此这个错误信息将
	通过 {{ form.non_field_errors }} 渲染。
####3.4 设置登录登出的跳转 URL
	在settings中设置以下参数：
	LOGOUT_REDIRECT_URL = '/index/'
	LOGIN_REDIRECT_URL = '/index/'
	表示登录后跳转到index页面。
###4. 注销和页面跳转
	当用户想切换登录账号，或者想退出登录状态时，这时候就需要注销已登录的账号。
	现在我们来为网站添加注销登录的功能，这个功能 Django 也已经为
	我们提供，我们只需做一点简单配置。
####4.1 注销登录
	注销登录的视图为 logout
####4.2 页面跳转
	对于一个网站来说，比较好的用户体验是登录、注册和注销后跳转回用户之前访问的页面。
	在登录和注销的视图函数中，Django 已经为我们处理了跳转回用户之前访问页面的流程。
	其实现的原理是，在登录和注销的流程中，始终传递一个 next 参数记录用户之前访问页面的 URL。
	我们需要做的就是在用户访问登录或者注销的页面时，在 URL 中传递一个 next 参数给视图函数，具体做法如下：
		<button class="btn btn-default">
		  <a href="{% url 'logout' %}?next={{ request.path }}">注销登录</a>
		</button>
		
		<button class="btn btn-default">
		  <a href="{% url 'login' %}?next={{ request.path }}">登录</a>
		</button>
	可以看到，我们在登录和注销的 URL 后加了 next 参数，其值为 {{ request.path }}。request.path 是用户当前访问页面的 URL。
	为了在整个登录流程中记录 next 的值，还需要在登录表单中增加一个表单控件，用于传递 next 值。
	registration/login.html
	
	<form class="form" action="{% url 'login' %}" method="post">
	  ...                 
	  <button type="submit" class="btn btn-primary btn-block">登录</button>
	  <input type="hidden" name="next" value="{{ next }}"/>
	</form>
	即在表单中增加了一个隐藏的 input 控件，其值为 {{ next }}，即之前通过 URL 参数传递给登录视图函数的，然后登录视图函数又将该值传递给了 login.html 模板。
	这样在整个登录流程中，始终有一个记录着用户在登录前页面 URL 的变量 next 在视图和模板间来回传递，直到用户登录成功后再跳转回 next 记录的页面。
	类似的，我们也希望用户注册后返回注册前页面。
	不过由于注册视图函数是我们自己写的，所以需要我们自己去获取跳转的地址。
	需要修改一下注册视图函数：
		def register(request):
		    redirect_to = request.POST.get('next', request.GET.get('next', ''))
		    if request.method == 'POST':
		        form = RegisterForm(request.POST)
		
		        if form.is_valid():
		            form.save()
		            if redirect_to:
		                return redirect(redirect_to)
		            else:
		                return redirect(reverse('blog:index'))
		    else:
		        form = RegisterForm()
		    return render(request, 'users/register.html', {'form':form,'next':redirect_to})
	然后修改模板
		base.html
			 <a href="{% url 'users:register' %}?next={{ request.path }}">注册</a>
###5. 修改密码
	修改密码的的视图函数默认渲染的模板名为 password_change_form.html
	首先在 registration/ 下新建一个password_change_form.html 文件，写入表单代码。代码几乎和登录界页面一样。
	base.html中增加修改密码的超链接，注意必须是登录才能显示这个超链接：
		<li class="cl-effect-11"><a href="{% url 'password_change' %}?next={{ request.path }}" data-hover="修改密码">修改密码</a></li>
	然后编写密码修改成功页面模板。
###6. 重置密码
	当用户不小心忘记了密码时，网站需要提供让用户找回账户密码的功能。	
	我们将发送一封含有重置用户密码链接的邮件到用户注册时的邮箱，用户点击收到的链接就可以重置他的密码。
####6.1 发送邮件设置
	Django 内置了非常方便的发送邮件的功能，不过需要在 settings.py 中做一些简单配置。
	生产环境下通常需要使用真实的邮件发送服务器，配置步骤会比较多一点。
	不过 Django 为开发环境下发送邮件提供了一些方便的 Backends 来模拟真实邮件的发送，例如直接发送邮件到终端。
	在 settings.py 中加入以下设置：
		EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
	这样 Django 将把邮件发送到终端。
####6.2 编写重置密码模板
	重置的视图函数默认渲染的模板名为 password_reset_form.html，因此首先在 registration/ 下新建一个 password_reset_form.html 文件.
	此外，修改一下重置密码按钮的超链接属性：
	registration/login.html
		...
		<div class="flex-left top-gap text-small">
		  <div class="unit-2-3"><span>没有账号？<a href="{% url 'users:register' %}">立即注册</a></span></div>
		  <div class="unit-1-3 flex-right"><span><a href="{% url 'password_reset' %}">忘记密码？</a></span></div>
		</div>
####6.2 编写邮件发送成功页面模板
	用户在重置密码页面输入注册时的邮箱后，Django 会把用户跳转到邮件发送成功页面.
	该页面渲染的模板为 password_reset_done.html，因此再添加一个密码修改成功页面的模板.
####6.3 编写设置新密码页面模板
	在接收到的重置密码邮件中有一个设置新密码的链接，点击该链接就会跳转到给账户设置新密码的页面，以便用户给已忘记密码的账户设置一个全新的密码。
	该页面渲染的模板为 password_reset_confirm.html	
####6.4 编写设置新密码成功页面模板
	用户在设置新密码页面输入新密码后，Django 会把用户跳转到设置新密码成功页面.
	该页面渲染的模板为 password_reset_complete.html，因此再添加一个设置新密码成功页面的模板.
###7 自定义认证后台
	Django auth 应用默认支持用户名（username）进行登录。
	但是在实践中，网站可能还需要邮箱、手机号、身份证号等进行登录。
	这就需要我们自己写一个认证后台，用于验证用户输入的用户信息是否正确，从而对拥有正确凭据的用户进行登录认证。
####7.1 Django 验证用户合法性的方式
	Django 对用户登录的验证工作均在一个被称作认证后台（Authentication Backend）的类中进行。
	这个类是一个普通的 Python 类，它有一个 authenticate 方法，
	接收登录用户提供的凭据（如用户名或者邮箱以及密码）作为参数，
	并根据这些凭据判断用户是否合法（即是否是已注册用户，密码是否正确等）。
	面是 Django 内置的认证后台的部分源代码：
		django.contrib.auth.backends
	
		class ModelBackend(object):
		    """
		    Authenticates against settings.AUTH_USER_MODEL.
		    """
		
		    def authenticate(self, request, username=None, password=None, **kwargs):
		        if username is None:
		            username = kwargs.get(UserModel.USERNAME_FIELD)
		        try:
		            user = UserModel._default_manager.get_by_natural_key(username)
		        except UserModel.DoesNotExist:
		            # Run the default password hasher once to reduce the timing
		            # difference between an existing and a non-existing user (#20760).
		            UserModel().set_password(password)
		        else:
		            if user.check_password(password) and self.user_can_authenticate(user):
		                return user
	这段代码根据用户传入的 username 和 password，验证该 username 对应的用户是否存在以及密码是否正确，是则返回该 user 对象。
	可以定义多个认证后台，Django 内部会逐一调用这些后台的 
	authenticate 方法来验证用户提供登录凭据的合法性，一旦
	通过某个后台的验证，表明用户提供的凭据合法，从而允许登录该用户。
####7.2 Email Backend
```python
因为 Django auth 应用内置只支持用户名和密码的认证方式，所以
目前用户是无法使用 Email 进行登录的。
为了实现邮箱登录，我们需要编写一个认证后台。
这个后台的作用便是验证用户提供的凭据（这里是邮箱以及密码）是合法的。
完全仿照内置的 ModelBackend 代码即可。	
首先在 users 应用下新建一个 backends.py 文件，然后写入如下代码：
	users/backends.py

	from .models import User
	
	class EmailBackend(object):
	    def authenticate(self, request, **credentials):
	        # 要注意登录表单中用户输入的用户名或者邮箱的 field 名均为 username
	        email = credentials.get('email', credentials.get('username'))
	        try:
	            user = User.objects.get(email=email)
	        except User.DoesNotExist:
	            pass
	        else:
	            if user.check_password(credentials["password"]):
	                return user
	
	    def get_user(self, user_id):
	        """
	        该方法是必须的
	        """
	        try:
	            return User.objects.get(pk=user_id)
	        except User.DoesNotExist:
	            return None
逻辑非常简单，就是根据用户提供的 Email 和密码，检查该 email
对应的用户是否存在，如果存在则检查密码是否正确，如果密码也没有
问题，则返回该 user 对象。
```
####7.3 配置 Backend
	接下来就要告诉 Django，需要使用哪些 Backends 对用户的凭据信息进行验证，这需要在 settings.py 中设置：
	settings.py
	
	AUTHENTICATION_BACKENDS = (
	    'django.contrib.auth.backends.ModelBackend',
	    'users.backends.EmailBackend',
	)
	第一个 Backend 是 Django 内置的 Backend，当用户提供的是用
	户名和正确的密码时该 Backend 会通过验证；第二个 Backend 是
	刚刚自定义的 Backend，当用户提供的是 Email 和正确的密码时该
	 Backend 会通过验证。

###8 使用modelform上传头像。
1. 服务器端安装pillow
   pip install pillow

2. 添加图片字段到文章models.py
  headshot = models.ImageField(upload_to='avatar/%Y/%m/%d/',default='default.jpg', verbose_name='头像')

3. settings.py 增加图片存储路径，同时创建目录。
  MEDIA_URL = '/uploads/'
  MEDIA_ROOT = os.path.join(BASE_DIR, 'uploads')

4. 项目urls.py添加如下static
  from django.conf import settings
  from django.conf.urls.static import static

  urlpatterns = [
              ........
          ] + static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)

5. form表单提交方法为post，且含有enctype="multipart/form-data"属性

6. view函数接收：
  form = RegisterForm(request.POST, request.FILES)

7. 在前台展示：

  <img src="{{ user.headshot.url }} " height="40" width="40"/>
