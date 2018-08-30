#项目day13 
###1. 自动生成文章摘要
	两种方法可以实现自动生成文章摘要。
	1）复写 save 方法
		通过复写模型的 save 方法，在数据被保存到数据库前，先从 body 字段摘取 N 个字符保存到 excerpt 字段中，从而实现自动摘要的目的。具体代码如下：
			    def save(self, *args, **kwargs):
		        # 如果没有填写摘要
		        if not self.excerpt:
					# 首先实例化一个 Markdown 类，用于渲染 body 的文本
		            md = markdown.Markdown(extensions=[
		                'markdown.extensions.extra',
		                'markdown.extensions.codehilite',
		            ])
					# 先将 Markdown 文本渲染成 HTML 文本 
					# strip_tags 去掉 HTML 文本的全部 HTML 标签
					# 从文本摘取前 54 个字符赋给 excerpt
		            self.excerpt = strip_tags(md.convert(self.content))[:54]
				# 调用父类的 save 方法将数据保存到数据库中
		        super(Post,self).save(*args, **kwargs)
	2）使用 truncatechars 模板过滤器
		<p>{{ post.content|truncatechars:54 }}...</p>
		不过这种方法的一个缺点就是如果前 54 个字符含有块级 HTML 元素标签的话（比如一段代码块），
		会使摘要比较难看。所以推荐使用第一种方法。

###2. 基于类的通用视图：ListView 和 DetailView
	在开发网站的过程中，有一些视图函数虽然处理的对象不同，但是其大致的代码逻辑是一样的。
	Django 把这些相同的逻辑代码抽取了出来，写成了一系列的通用视图函数，即基于类的通用视图（Class Based View）。
	使用类视图是 Django 推荐的做法，而且熟悉了类视图的使用方法后，能够减少视图函数的重复代码，节省开发时间。
	接下来就让我们把博客应用中的视图函数改成基于类的通用视图。
####2.1 ListView
		在我们的博客应用中，有几个视图函数是从数据库中获取文章（Post）列表数据的：
			【blog/views.py】 
			def index(request):
				# ...
			def archives(request, year, month): 
				# ... 
			def category(request, pk):
				# ...
		这些视图函数都是从数据库中获取文章（Post）列表，
		唯一的区别就是获取的文章列表可能不同。比如 index 获取全部文章列表，category 获取某个分类下的文章列表。
		针对这种从数据库中获取某个模型列表数据（比如这里的 Post 列表）的视图，Django 专门提供了一个 ListView 类视图。
		1）将 index 视图函数改写为类视图
			class IndexView(ListView):
			    model = Post
			    template_name = 'blog/index.html'
			    context_object_name = 'post_list'
			要写一个类视图，首先需要继承 Django 提供的某个类视图。
			至于继承哪个类视图，需要根据你的视图功能而定。比如获取列表可以基础ListView，获取详细信息可以基础DetailView。
			然后通过一些属性来指定这个视图函数需要做的事情。。这里我们指定了三个属性。
			model：将 model 指定为 Post，告诉 Django 我要获取的模型是 Post。
			template_name：指定这个视图渲染的模板。
			context_object_name：指定获取的模型列表数据保存的变量名。这个变量会被传递给模板。
			在 URL 配置中把 index 视图替换成类视图 IndexView：
				【blog/urls.py】
				 app_name = 'blog' 
				 urlpatterns = [ url(r'^$', views.IndexView.as_view(), name='index'), ... ]
		2）将 category 视图函数改写为类视图
			category获取的是某个分类下的全部文章，ListView默认的行为已经满足不了我们的需求。我们通过重写父类的get_queryset方法来定制我们的需求。
			在类视图中，从 URL 捕获的关键词参数值保存在实例的 kwargs 属性（是一个字典）里，
			非关键词参数值保存在实例的 args 属性（是一个列表）里。
			class CategoryView(ListView):
			    model = Post
			    template_name = 'blog/index.html'
			    context_object_name = 'post_list'
			
			    def get_queryset(self):
			        cate = get_object_or_404(Category,pk=self.kwargs.get('pk'))
			        return super(CategoryView, self).get_queryset().filter(category=cate)
			可以看到 CategoryView 类中指定的属性值和 IndexView 中是一模一样的。
			所以如果为了进一步节省代码，甚至可以直接继承 IndexView：
				class CategoryView(IndexView):
				    def get_queryset(self):
				        cate = get_object_or_404(Category,pk=self.kwargs.get('pk'))
				        return super(CategoryView, self).get_queryset().filter(category=cate)
		3）将 archives 视图函数改写成类视图
			方法一样：
				class ArchivesView(IndexView):
				    def get_queryset(self):
				        return super(ArchivesView,self).get_queryset().filter(
				            created_time__year=self.kwargs.get('year'),
				            created_time__month=self.kwargs.get('month')
				        )
			url：
				url(r'^category/(?P<pk>\d+)/$', views.CategoryView.as_view(), name='category'),
	
	2）DetailView
		除了从数据库中获取模型列表的数据外，从数据库获取模型的一条记录也是常见的需求。
		比如查看某篇文章的详情，就是从数据库中获取这篇文章的记录然后渲染模板。
		对于这种类型的需求，Django 提供了一个 DetailView 类视图。	
		下面将 detail 视图函数转换为等价的类视图 PostDetailView。
			class PostDetailView(DetailView):
			    model = Post
			    template_name = 'blog/detail.html'
			    context_object_name = 'post'
			
			    # 复写get方法
			    def get(self, request, *args, **kwargs):
			        # get方法返回的是一个HttpResponse实例
			        # 先调用父类的get方法，self.object属性中才会有Post模型的实例。
			        response = super(PostDetailView,self).get(request, *args, **kwargs)
			        # 文章阅读量+1
			        # self.object的值就是被访问的文章对象post
			        self.object.increase_views()
			        # 视图函数必须返回一个HttpResponse对象
			        return response
			
			    def get_object(self, queryset=None):
			        # 复写get_object方法的目的是因为需要对post对象本身的body值进行渲染
			        post = super(PostDetailView, self).get_object(queryset=None)
			        post.content = markdown.markdown(post.body,extensions=[
			            'markdown.extensions.extra',
			            'markdown.extensions.codehilite',
			            'markdown.extensions.toc'
			        ])
			        return post
			    
			    def get_context_data(self, **kwargs):
			        # 复写get_context_data的目的是因为除了将post传递给模板之外（DetailView已经帮我们做了）
			        # 还要把评论表单，post下的评论列表传递给模板。也就是往context中填充内容。
			        # 调用父类的get_context_data,先在context中生成self.object即Post的实例对象
			        context = super(PostDetailView, self).get_context_data(**kwargs)
			        form = CommentForm()
			        comment_list = self.object.comment_set.all()
			        context.update({
			            'form': form,
			            'comment_list': comment_list
			        })
			        return context
			PostDetailView略复杂，因为原本的detail函数也不简单。下面来详细讲解一下PostDetailView。
			复写get方法，对应着 detail 视图函数中将 post 的阅读量 +1 的那部分代码。每次调用视图函数都会执行get方法的代码。
			复写get_object方法，对应着 detail 视图函数中根据文章的 id（也就是 pk）获取文章，
			然后对文章的 post.body 进行 Markdown 渲染的代码部分。
			get_object是默认的查找单个对象的方法，我们修改了默认的行为。
			最后我们复写了get_context_data方法。这部分对应着 detail 视图函数中
			生成评论表单、获取 post 下的评论列表的代码部分。
			这个方法返回的值是一个字典，这个字典就是模板变量字典，最终会被传递给模板。
			简单来说，如果我们想要每次调用detail函数的时候做一些事情，就复写get方法。
					如果想要修改默认的查询行为，复写get_object方法。
					如果想要添加额外的context变量，复写get_context_data方法。
**在类视图中，从 URL 捕获的关键词参数值保存在实例的 kwargs 属性（是一个字典）里，非关键词参数值保存在实例的 args 属性（是一个列表）里。**

# 先调用父类的get方法，self.object属性中才会有Post模型的实例

###3. Django Pagination 简单分页

	使用Django 内置的 Pagination 能够帮助我们实现简单的分页功能。
####3.1 Paginator 类的常用方法
	# 对 item_list 进行分页，每页包含 2 个数据。 
		>>> item_list = ['john', 'paul', 'george', 'ringo'] 
		>>>> p = Paginator(item_list, 2)	
		# 取第 2 页的数据 
		>>> page2 = p.page(2) 
		>>> page2.object_list 
		['george', 'ringo']
		#查询特定页的当前页码数
		>>> page2.number
			2
		#查看分页后的总页数
		>>> p.num_pages		
		2
		#查看某一页是否还有上一页
		>>> page2.has_previous()
		True
		# 查询第二页上一页的页码
		>>> page2.previous_page_number()
		1
		#查询第二页是否还有下一页
		>>> page2.has_next()
		False
		# 查询第二页下一页的页码
		>>> page2.next_page_number()
		django.core.paginator.EmptyPage: That page contains no results
####3.2 用 Paginator 给文章列表分页
	视图函数中使用分页器的一般套路：
		from django.core.paginator import Paginator, EmptyPage, PageNotAnInteger
		from django.shortcuts import render
		
		def listing(request):
		    contact_list = Contacts.objects.all()
			#把要分页的对象列表，和每页要显示多少条数据传给Paginator，生成一个分页器的实例对象 。
		    paginator = Paginator(contact_list, 25) # Show 25 contacts per page
			#page对象代表被分出来的每一页。通过页码来请求获得页面。
		    page = request.GET.get('page')
		    try:
		        contacts = paginator.page(page)
		    except PageNotAnInteger:
		        # 如果请求的号码不是整数，那么返回第一页。
		        contacts = paginator.page(1)
		    except EmptyPage:
		        # 如果请求的页码超过了最大页数，那么返回最后一页。
		        contacts = paginator.page(paginator.num_pages)
		
		    return render(request, 'list.html', {'contacts': contacts})
	在我们的类视图ListView中已经帮我们写好了上述的分页逻辑，
	我们只需通过指定 paginate_by 属性来开启分页功能即可。即在代码中：
		# 指定 paginate_by 属性后开启分页功能，其值代表每一页包含多少篇文章 
		paginate_by = 10
####3.3 在模板中设置分页导航
```python
ListView 传递了以下和分页有关的模板变量供我们在模板中使用：
	paginator ，即 Paginator 的实例。
	page_obj ，当前请求页面分页对象。
	is_paginated，是否已分页。只有当分页后页面超过两页时才算已分页。
	object_list，请求页面的对象列表，和 post_list 等价。所以在模板中循环文章列表时可以选 post_list ，也可以选 object_list。
在模板中使用：
	{% if is_paginated %}
        <div class="pagination-simple">
            {% if page_obj.has_previous %}
                <a href="?page={{ page_obj.previous_page_number }}">上一页</a>
            {% endif %}
            <span class="current">第{{ page_obj.number }}页/共{{ paginator.num_pages }}页</span>
            {% if page_obj.has_next %}
                <a href="?page={{ page_obj.next_page_number }}">下一页</a>
            {% endif %}
        </div>
     {% endif %}
```

####3.3 在模板中设置分页导航
```python
需求：设置类似百度的的分页。
核心其实只需要把页码列表找出来，前台循环显示页码数即可。
单独定义一个处理分页的函数，放到utils中，作为一个工具函数。
通过分析分页的情况可以分为1种特殊情况，3中普通情况来处理。
特殊情况为总页数小于每个页面最大显示的页码数。
普通情况为当前页在头部，中部和尾部的情况。
	utils.py
	import math

    def custompaginator(num_pages, current_page, max_page):
        middle = math.ceil(max_page / 2)
        # 特殊情况，最大页数小于最大显示的页数
        if num_pages < max_page:
            start = 1
            end = num_pages
        else:
            # 当前页小于等于middle 时。
            if current_page <= middle :
                start = 1
                end = max_page
            else:
                # 中间情况
                start = current_page - middle
                end = current_page + middle - 1
                # 当前页在尾巴的情况
                if current_page + middle > num_pages:
                    start = num_pages - max_page + 1
                    end = num_pages
        return start,end
	views.py
	class IndexView(ListView):
	    model = Post
	    template_name = 'blog/index.html'
	    context_object_name = 'post_list'
	    paginate_by = 2
	
	    # 复写该方法以便我们能够插入自定义的模板变量
	def get_context_data(self, **kwargs):
        # 先调用父类的方法，获取默认的context
        context = super().get_context_data(**kwargs)
        paginator = context.get('paginator')
        page = context.get('page_obj')
        start, end = utils.custompaginator(paginator.num_pages, page.number, 4)
        context.update({
            'page_range': range(start, end + 1)
        })
        return context
 前端分页代码：
	{% if is_paginated %}
        <div class="pagination-simple">
            {% if page_obj.has_previous %}
                <a href="?page={{ page_obj.previous_page_number }}">上一页</a>
            {% endif %}

            {% for i in page_range %}
                {% if page_obj.number == i %}
                    <a style="color:red;font-size:26px;padding: 5px" href="?page={{ i }}">{{ i }}</a>
                {% else %}
                    <a style="padding: 5px" href="?page={{ i }}">{{ i }}</a>
                {% endif %}
            {% endfor %}
            {% if page_obj.has_next %}
                <a href="?page={{ page_obj.next_page_number }}">下一页</a>
            {% endif %}
        </div>
    {% endif %}
```


###4. 几个改进

	在模型中指定排序
		在post模型中添加meta类，定义默认的排序顺序
			class Meta: 
				ordering = ['-created_time']
	完善跳转链接
		修改 Logo 处的超链接
			<h1><a href="{% url 'blog:index' %}"><b>千峰博客</b></a></h1>
		导航栏的首页
			<li class="cl-effect-11"><a href="{% url 'blog:index' %}" data-hover="首页">首页</a></li>
		评论及评论数：
			<span class="comments-link"><a href="{{ post.get_absolute_url }} ">{{ post.comment_set.count }} 评论</a></span>
	点击分类跳转到分类页面
		index.html中
			<span class="post-category"><a href="{% url 'blog:category' post.category.pk %}">{{ post.category }}</a></span>
	点击评论量就跳转到文章详情页的评论处
			<span class="comments-link"><a href="{{ post.get_absolute_url }}#comment-area">{{ post.comment_set.count }} 评论</a></span>

###5. 统计各个分类下的文章数
	最优雅的方式就是使用 Django 模型管理器的 annotate 方法。
	在blog_tags.py中
		@register.simple_tag
		def get_categories():
		    return Category.objects.annotate(num_posts=Count('post')).filter(num_posts_gt=0)
	
	在模板中引用新增的属性
		<span class="post-count">({{ category.num_posts }})</span>