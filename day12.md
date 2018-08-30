#项目day12  
###1. 支持 Markdown 语法和代码高亮
####1.1 安装 Python Markdown
	pip install markdown
	或者在pycharm中安装markdown
####1.2 在 detail 视图中渲染 Markdown
```python
将 Markdown 格式的文本渲染成 HTML 文本非常简单，只需调用这个库的 markdown 方法即可。
【blog/views.py】 
import markdown 
from django.shortcuts import render, get_object_or_404 
from .models import Post 
def detail(request, pk): post = get_object_or_404(Post, pk=pk) 
# 相当于对post.content多了一个中间步骤，先将 Markdown 格式的文本渲染成 HTML 文本再传递给模板 
post.content = markdown.markdown(post.content, 
extensions=[ 
'markdown.extensions.extra',
'markdown.extensions.codehilite',
'markdown.extensions.toc', ]) 
return render(request, 'blog/detail.html', context={'post': post})
在admin中发布一篇markdown语法书写的测试博客。
	# 一级标题
	## 二级标题
	### 三级标题
	
	>引用
	
	- 无序列表
	- 无序列表
	- 无序列表
	
	[百度的链接](http://baidu.com)
	
	![图片](http://box.bdimg.com/static/fisp_static/common/img/searchbox/logo_news_276_88_1f9876a.png)
	
	```python
	def detail(request, pk):
	    post = get_object_or_404(Post, pk=pk)
	    post.body = markdown.markdown(post.body,
	                                  extensions=[
	                                      'mardown.extensions.extra',
	                                      'markdown.extensions.codehilite',
	                                      'markdown.extensions.toc',
	                                  ])
	    return render(request, 'blog/detail.html', locals())
	```

博客发布之后，发现markdown语法并不起作用，给post.body加上safe标签。
{{ post.body |safe }}
```
####1.3 代码高亮
	1) 安装 Pygments
		激活虚拟环境，pip install Pygments，或者在pycharm中安装。
	2）引入样式文件
		base.html中加入<link rel="stylesheet" href="{% static 'blog/css/highlights/github.css' %}">
		重启服务器看看效果。
###2. 页面侧边栏：使用自定义模板标签
####2.1 使用自定义模板标签
	1）在应用目录下新建templatetags目录。
	2）在templatetags目录中新建__init__.py文件
	3）新建自定义tag代码所在的py文件。比如blog_tags.py
	4) 在blog_tags.py中开头写上这两句：
		from django import template
		register = template.Library()
	5）在想要变成自定义tag的函数前加上@register.simple_tag，比如：
		@register.simple_tag 
		def get_recent_posts(num=5): 
			return Post.objects.all().order_by('-created_time')[:num]
	6）在模板中使用自定义tag：
		在模板中导入自定义tag文件：{% load blog_tags %}
		使用的写法：{% get_recent_posts as recent_post_list %}
####2.2 使用自定义模板标签实现页面侧边栏
```python
from django import template
from ..models import Post, Category

register = template.Library()

#最新文章模板标签
@register.simple_tag
def get_recent_posts(num=5):
    return Post.objects.all().order_by('-created_time')[:num]

#归档模板标签
@register.simple_tag
def archives():
    return Post.objects.dates('created_time','month',order='DESC')
#分类模板标签
@register.simple_tag
def get_categories():
    return Category.objects.all()
修改base.html
	最新文章：
		<h3 class="widget-title">最新文章</h3>
            {% get_recent_posts as recent_post_list %}
            <ul>
                {% for post in recent_post_list %}
                    <li>
                        <a href="{{ post.get_absolute_url }}">{{ post.title }}</a>
                    </li>
                {% empty %}
                    暂无文章
                {% endfor %}
            </ul>
	归档：
		<h3 class="widget-title">归档</h3>
            {% archives as date_list %}
            <ul>
               {% for date in date_list %}
                   <li>
                    <a href="#">{{ date.year }}年{{ date.month }}月</a>
                   </li>
                {% empty %}
                   暂无归档
                {% endfor %}
            </ul>
	分类：
		<h3 class="widget-title">分类</h3>
            {% get_categories as category_list %}
            <ul>
                {% for category in category_list %}
                    <li>
                        <a href="#">{{ category.name }}<span class="post-count">(13)</span></a>
                    </li>
                {% empty %}
                    暂无分类
                {% endfor %}
            </ul>
```
###3. 分类与归档
####3.1 归档
		使用filter获取归档数据。blog视图文件中增加如下视图函数：
		def archives(request, year, month):
		    post_list = Post.objects.filter(created_time__year=year,
		                                    created_time__month=month,
		                                    ).order_by('-created_time')
		    return render(request,'blog/index.html', locals())
		blog urls.py中增加下面一行：
			url(r'^archives/(?P<year>[0-9]{4})/(?P<month>[0-9]{1,2})/$', views.archives, name='archives'),
		修改base模板中归档部分的超链接
			<li>
	        	<a href="{% url 'blog:archives' date.year date.month %}">{{ date.year }}年{{ date.month }}月</a>
	       </li>
####3.2 分类
	blog视图文件中添加如下视图函数：
		def category(request,pk):
		    cate = get_object_or_404(Category,pk=pk)
		    post_list = Post.objects.filter(category=cate).order_by('-created_time')
		    return render(request,'blog/index.html', {'post_list':post_list})
	url中添加如下：
		url(r'^category/(?P<pk>[0-9]+)/$', views.category, name='category'),
	base中分类部分的超链接修改如下：
		<li>
	    	<a href="{% url 'blog:category' category.pk %}">{{ category.name }}<span class="post-count">(13)</span></a>
		</li>
###4. 评论
####4.1 创建评论应用
	相对来说，评论其实是另外一个比较独立的功能。Django 提倡，如果功能相对比较独立的话，最好是创建一个应用，把相应的功能代码写到这个应用里。
	python manage.py startapp comments
####4.2 设计评论的数据库模型
	class Comment(models.Model):
	    name = models.CharField(max_length=100)
	    email = models.EmailField(max_length=255)
	    url = models.URLField(blank=True)
	    text = models.TextField()
	    created_time = models.DateTimeField(auto_now_add=True)
	    post = models.ForeignKey('blog.Post')
	
	    def __str__(self):
	        return self.text[:20]
	生成完model后记得执行数据迁移。
####4.3 评论表单设计
	在comments app目录下新建forms.py文件，该文件中写入以下代码：
	使用modelform
		from django import forms
		from .models import Comment
				
		class CommentForm(forms.ModelForm):
		    class Meta:
		        model = Comment
		        fields = ['name', 'email', 'url','text']
####4.4 评论视图函数
```python
from django.shortcuts import render, get_object_or_404, redirect
from blog.models import Post
from .models import Comment
from .forms import CommentForm
# Create your views here.

def post_comment(request, post_pk):
    post = get_object_or_404(Post, pk=post_pk)
    if request.method == 'POST':
        form = CommentForm(request.POST)
        if form.is_valid():
            comment = form.save(commit=False)
            comment.post = post
            comment.save()
            return redirect(post)
        else:
            comment_list = post.comment_set.all()
            context = {
                'post':post,
                'form':form,
                'comment_list':comment_list
            }
            return render(request, 'blog/detail.html', context=context)
    return redirect(post)
```

####4.5 绑定 URL
	app_name = 'comments' 
	urlpatterns = [ url(r'^comment/post/(?P<post_pk>[0-9]+)/$', views.post_comment, name='post_comment'), ]
	最后别忘了在项目的urls中包含comments的url
		url(r'', include('comments.urls')),
####4.6 更新文章详情页面的视图函数
	在detail视图函数中加上这两句，让detail视图函数获取post所属的评论列表。：
		form = CommentForm()
		comment_list = post.comment_set.all()
####4.7 在前端渲染表单
	修改前端评论的form表单部分代码：
		<form action="{% url 'comments:post_comment' post.pk %}" method="post" class="comment-form">
	    {% csrf_token %}
	    <div class="row">
	        <div class="col-md-4">
	            <label for="{{ form.name.id_for_label }}">名字：</label>
	            {{ form.name }}
	            {{ form.name.errors }}
	        </div>
	        <div class="col-md-4">
	            <label for="{{ form.email.id_for_label }}">邮箱：</label>
	            {{ form.email }}
	            {{ form.email.errors }}
	        </div>
	        <div class="col-md-4">
	            <label for="{{ form.url.id_for_label }}">网址：</label>
	            {{ form.url }}
	            {{ form.url.errors }}
	        </div>
	        <div class="col-md-12">
	            <label for="{{ form.text.id_for_label }}">评论：</label>
	            {{ form.text }}
	            {{ form.text.errors }}
	            <button type="submit" class="comment-btn">发表</button>
	        </div>
	    </div>    <!-- row -->
	    </form>
####4.8 显示评论内容
	在detail.html中删掉占位用的评论内容的 HTML 代码，使用如下代码替换：
		<ul class="comment-list list-unstyled">
	        {% for comment in comment_list %}
	        <li class="comment-item">
	            <span class="nickname">{{ comment.name }}</span>
	            <time class="submit-date" >{{ comment.created_time }}</time>
	            <div class="text">
	                {{ comment.text }}
	            </div>
	        </li>
	        {% empty %}
	            暂无评论
	        {% endfor %}
	    </ul>
###6. 统计文章阅读量
####6.1 修改模型
	1) 增加新字段
		Post中增加：
		views = models.PositiveIntegerField(default=0)
	2）增加模型方法
		def increase_views(self): 
			self.views += 1 
			self.save(update_fields=['views'])
	3）迁移数据库
####6.2 修改视图函数
	在detail视图函数中，在post = get_object_or_404(Post, pk=pk)后增加一行：
		post.increase_views()
####6.3 在模板中显示阅读量
		index.html
			<span class="views-count"><a href="{{ post.get_absolute_url }}">{{ post.views }} 阅读</a></span>
		detail.html
			<span class="views-count"><a href="#">{{ post.views }} 阅读</a></span> 
