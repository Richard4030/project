#项目day11
###1. 企业常见开发模式
	1.1 瀑布式开发
		瀑布模型式是最典型的预见性的方法，严格遵循预先计划的需求分析、设计、编码、集成、测试、维护的步骤顺序进行。
		瀑布式的主要的问题是它的严格分级导致的自由度降低，项目早期即作出承诺导致对后期需求的变化难以调整，
		代价高昂。瀑布式方法在需求不明并且在项目进行过程中可能变化的情况下基本是不可行的。
	1.2 迭代式开发 （目前公司用的较多的开发模式）
		每次只设计和实现这个产品的一部分；
		逐步逐步完成的方法叫迭代开发；
		每次设计和实现一个阶段叫做一个迭代.
		在迭代式开发方法中，整个开发工作被组织为一系列的短小的、
		固定长度（如3周）的小项目，被称为一系列的迭代。
		每一次迭代都包括了需求分析、设计、实现与测试。
	1.3 螺旋开发
		“螺旋模型”刚开始规模很小，当项目被定义得更好、更稳定时，逐渐展开。
		螺旋模型很大程度上是一种风险驱动的方法体系，因为在每个阶段之前及经常发生的循环之前，都必须首先进行风险评估。
		步骤：
			（1）制定计划：确定软件目标，选定实施方案，弄清项目开发的限制条件；
			（2）风险分析：分析评估所选方案，考虑如何识别和消除风险；
			（3）实施工程：实施软件开发和验证；
			（4）客户评估：评价开发工作，提出修正建议，制定下一步计划。
	1.4 敏捷开发 （比较热门的开发模式）
		和迭代式开发类似，敏捷开发的周期可能更短，并且更加强调队伍中的高度协作。一个小功能叫做一个story。开发人员要完成stroy文档的编写。

###2. 软件开发的一般流程
	需求分析及确认：由需求分析工程师与客户确认甚至挖掘需求。输出需求说明文档。
	概要设计及详细设计：开发对需求进行概要设计，包括系统的基本处理流程，组织结构、模块划分、接口设计、数据库结构设计等。然后在概要设计的基础上进行详细设计。详细设计中描述实现具体模块所涉及到的主要算法、数据结构、类的层次结构及调用关系，需要说明软件系统各个层次中的每一个程序(每个模块或子程序)的设计考虑，以便进行编码和测试。基本达到伪代码的层面。
	编码：根据详细设计文档进行编码。在实际的项目开发中，编码是占时间最少的。
	测试：一般有专业测试团队进行测试。
	发布或上线：提供各种文档，比如杀毒软件扫描文档，安装手册，操作指南等一系列文档资料打包与程序一起发布。
	当然后续还会有验收和维护等操作。
###3. 博客项目的需求分析
	博客列表展示 
	博客详细信息展示
	最新文章
	归档
	分类
	标签云
	rss订阅
	搜索
###4. 设计博客项目模型
	帖子 post (title,content,excerpt,created_time,modified_time,tag,category,author,comment)
	标签 tag  (name)
	类别 category (name)
	作者 author   (name,password,email...)
	评论 comment  (content, created_time,author)
###5. 创建工程和app
	创建虚拟环境：virtualenv blogenv
	创建工程：django-admin startproject myblog
	修改时区和语言
	LANGUAGE_CODE = 'zh-hans'
	TIME_ZONE = 'Asia/Shanghai'
	创建app,并注册。
	python manage.py startapp blog
###6. 定义models
```python
根据分析好的models，编写models代码
from django.db import models
from django.contrib.auth.models import User
	#Create your models here.

class Category(models.Model):
    name = models.CharField(max_length=30)

    def __str__(self):
        return self.name
```


```python
class Tag(models.Model):
    name = models.CharField(max_length=30)

    def __str__(self):
        return self.name
```


```python
class Post(models.Model):
    title = models.CharField(max_length=100)
    content = models.TextField()

    created_time = models.DateTimeField(auto_now_add=True)
    modified_time = models.DateTimeField(auto_now=True)

    #摘要
    excerpt = models.CharField(max_length=200,blank=True, null=True)
    category = models.ForeignKey('Category')
    tags = models.ManyToManyField(Tag, blank=True)
    author = models.ForeignKey(User)

    def __str__(self):
        return self.title
    
```
###7. 数据迁移
	python manage.py makemigrations
	python manage.py migrate
###8. 创建第一个视图函数博客首页index
```python
创建第一个视图函数博客首页index
处理静态文件。
通过django admin后台发布文章
创建超级用户： 
注册models
	admin.site.register(Post)
	admin.site.register(Category)
	admin.site.register(Tag)
定制admin后台：
	class PostAdmin(admin.ModelAdmin):
	    list_display = ['title', 'created_time', 'modified_time', 'category', 'author']
	
	admin.site.register(Post, PostAdmin)
使用admin后台填充数据。
```
###9. 完成真正的博客首页
```python
def index(request):
    post_list = Post.objects.all().order_by('-created_time')
    return render(request,'blog/index.html',context={'post_list':post_list})
修改index.html
	删掉：
		<article class="post post-1"> ... </article> 
		<article class="post post-2"> ... </article> 
		<article class="post post-3"> ... </article>
		...
	改成：
		{% for post in post_list %} 
			<article class="post post-{{ post.pk }}"> ... </article> 
		{% empty %} 
			<div class="no-post">暂时还没有发布的文章！</div>
		{% endfor %}
	再把相关的地方都替换掉。
设置url：
	from django.conf.urls import url 
	from . import views 
	urlpatterns = [ url(r'^$', views.index, name='index'), ]
项目url：
	from django.conf.urls import url, include
	from django.contrib import admin

	urlpatterns = [
	    url(r'^admin/', admin.site.urls),
	    url(r'',include('blog.urls')),
	]
```
###10. 博客文章详情页
```python
设计文章详情页的 URL：
	from django.conf.urls import url 
	from . import views 
	app_name = 'blog'
	urlpatterns = [ 
		url(r'^$', views.index, name='index'), 
		url(r'^post/(?P<pk>[0-9]+)/$', views.detail, name='detail'), ]
自定义get_absolute_url方法，用于获取post的url
	def get_absolute_url(self): 
		return reverse('blog:detail', kwargs={'pk': self.pk})
编写detail视图函数：
	def detail(request, pk): 
		post = get_object_or_404(Post, pk=pk) 
		return render(request, 'blog/detail.html', context={'post': post})
编写详情页模板：
	1）设置基础模板base.html
		<div class="content-body"> 
		<div class="container"> 
		<div class="row">
		 <main class="col-md-8"> 
		{% block main %} 
		{% endblock main %} 
		</main> 
		<aside class="col-md-4">
		{% block toc %} 
		{% endblock toc %} 
		<div class="widget widget-recent-posts">
	2）index模板
		{% extends 'blog/base.html' %}

		{% block main %}
		{% for post in post_list %}
		<article class="post post-{{ post.pk }}">
		    <header class="entry-header">
		        <h1 class="entry-title">
		            <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
		        </h1>
		        <div class="entry-meta">
		            <span class="post-category"><a href="{% url 'blog:category' post.category.pk %}">{{ post.category.name }}</a></span>
		            <span class="post-date"><a href="#"><time class="entry-date" datetime="{{ post.created_time }}">{{ post.created_time }}</time></a></span>
		            <span class="post-author"><a href="#">{{ post.author }}</a></span>
		            <span class="comments-link"><a href="{{ post.get_absolute_url }}#comment-area">{{ post.comment_set.count }} 评论</a></span>
		            <span class="views-count"><a href="#">588 阅读</a></span>
		        </div>
		    </header>
		    <div class="entry-content clearfix">
		        <p>{{ post.excerpt }}</p>
		        <div class="read-more cl-effect-14">
		            <a href="{{ post.get_absolute_url }}" class="more-link">继续阅读 <span class="meta-nav">→</span></a>
		        </div>
		    </div>
		</article>
		{% empty %}
		<div class="no-post">暂时还没有发布的文章！</div>
		{% endfor %}
		
		<div class="pagination">
		    <ul>
		        <li><a href="">1</a></li>
		        <li><a href="">...</a></li>
		        <li><a href="">4</a></li>
		        <li><a href="">5</a></li>
		        <li class="current"><a href="">6</a></li>
		        <li><a href="">7</a></li>
		        <li><a href="">8</a></li>
		        <li><a href="">...</a></li>
		        <li><a href="">11</a></li>
		    </ul>
		</div>
		{% endblock main %}
	3）detail模板
		{% extends 'blog/base.html' %}

		{% block main %}
		<article class="post post-{{ post.pk }}">
		    <header class="entry-header">
		        <h1 class="entry-title">{{ post.title }}</h1>
		        <div class="entry-meta">
		            <span class="post-category"><a href="#">{{ post.category.name }}</a></span>
		            <span class="post-date"><a href="#"><time class="entry-date" datetime="{{ post.created_time }}">{{ post.created_time }}</time></a></span>
		            <span class="post-author"><a href="#">{{ post.author }}</a></span>
		            <span class="comments-link"><a href="#">4 评论</a></span>
		            <span class="views-count"><a href="#">588 阅读</a></span>
		        </div>
		    </header>
		    <div class="entry-content clearfix">
		        {{ post.body |safe}}
		    </div>
		</article>
		<section class="comment-area" id="comment-area">
		    <hr>
		    <h3>发表评论</h3>
		    <form action="#" method="post" class="comment-form">
		        <div class="row">
		         	...
		            </div>
		        </div>
		    </form>
		    <div class="comment-list-panel">
			...
		    </div>
		</section>
		{% endblock main %}
```

​						