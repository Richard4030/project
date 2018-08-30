#项目day14
###1. 标签云
####1.1 获取标签列表
	标签云的实现方式和分类列表完全一样。
	定义一个get_tags模板标签，获取到文章数大于 0 的标签列表，然后在模板中渲染显示它。
		@register.simple_tag 
		def get_tags(): 
			# 记得在顶部引入 Tag model 
			return Tag.objects.annotate(num_posts=Count('post')).filter(num_posts__gt=0)
	
	然后在模板中循环显示这些标签：
		<div class="widget widget-tag-cloud">
	        <h3 class="widget-title">标签云</h3>
	        {% get_tags as tag_list %}
	        <ul>
	           {% for tag in tag_list %}
	               <li>
	                <a href="#">{{ tag.name }}</a>
	               </li>
	            {% empty %}
	               暂无标签！
	            {% endfor %}
	        </ul>
	    </div>
####1.2 显示某个标签下的文章列表
	1）标签视图函数
	代码和CategoryView几乎一样：
	class TagView(IndexView):
	    def get_queryset(self):
	        tag = get_object_or_404(Tag, pk=self.kwargs.get('pk'))
	        return super().get_queryset().filter(tags=tag)
	2）绑定 URL
	URL 模式和分类也是完全类似的：
		url(r'^tag/(?P<pk>\d+)/$', views.TagView.as_view(), name='tag'),
	3）设置标签跳转链接
	设置一下标签的超链接，这样点击标签后就可以跳转到该标签下的文章列表页面了。
		<a href="{% url 'blog:tag' tag.pk %}">{{ tag.name }}</a>
####1.3 在文章详情页显示标签
	需求：当在文章详情页面时，我们希望显示的这篇文章所属的标签。
	思路：通过post去查post所属的标签，那就是正向查找。直接通过post.tags.all()即可获得post下的标签列表。
		<div class="entry-content clearfix">
	        {{ post.body |safe }}
	        <div class="widget-tag-cloud">
	            <ul>
	                标签：
	                {% for tag in post.tags.all %}
	                    <li><a href="{% url 'blog:tag' tag.pk %}"># {{ tag.name }}</a></li>
	                {% endfor %}
	            </ul>
	
	        </div>
	    </div>
###2. RSS 订阅
	博客提供 RSS 订阅应该是标配。这样读者就可以通过一些聚合阅读工具来订阅你的博客，时时查看是否有更新，而不必每次都跳转到博客上来查看。
####2.1 RSS 简介
	RSS（Really Simple Syndication）是一种描述和同步网站内容的格式。，它采用 XML 作为内容传递的格式。
	简单来说就是网站可以把内容包装成符合 RSS 标准的 XML 格式文档。
	一旦网站内容符合一个统一的规范，那么人们就可以开发一种读取这种规范化的 XML 文档的工具来聚合各大网站的内容。
####2.2 使用 Django Feed 类
```python
根据以上对 RSS 的介绍，我们可以发现关键的地方就是根据网站的内容生成规范化的 XML 文档。
Django 已经内置了一些生成这个文档的方法.
在 blog 应用的根目录下（models.py 所在目录）新建一个 feeds.py 文件以存放和 RSS 功能相关的代码.然后在feeds.py中写入如下代码：
	from django.contrib.syndication.views import Feed

	from .models import Post
	
	class AllPostRssFeed(Feed):
	    # 显示在聚合阅读器上的标题
	    title = "千峰博客"
	
	    # 通过聚合阅读器跳转到网址的地址
	    link = '/index/'
	
	    # 显示在聚合阅读器上的描述信息
	    description = "千峰博客项目演示测试"
	
	    # 需要显示的内容条目
	    def items(self):
	        return Post.objects.all()
	
	    # 聚合器中显示的内容条目的标题
	    def item_title(self, item):
	        return '[%s]%s' % (item.category, item.title)
	
	    # 聚合器中显示的内容条目的描述
	    def item_description(self, item):
	        return item.content
以上代码就是指定要生成的xml文档内容。
```
####2.3 添加 URL
	接下来就是指定 URL 模式，让人们访问这个 URL 后就可以看到 Feed 生成的内容。
	通常 RSS 的 URL 配置直接写在项目的 urls.py 文件里。
	项目的urls.py中：
		记得在顶部引入 AllPostsRssFeed
		url(r'^all/rss/$', AllPostRssFeed(), name='rss'),
**通常 RSS 的 URL 配置直接写在项目的 urls.py 文件里**

####2.4 修改模板

	简单修改一下基模板，把 RSS 的 URL 添加到模板中，放在标签云下面：
	【templates/base.html】
		<div class="rss">
	        <a href="{% url 'rss' %}"><span class="ion-social-rss-outline"></span> RSS 订阅</a>
	    </div>	
####2.5 RSS 测试插件
	使用360浏览器，安装一个RSS Feed Reader应用。
	订阅我们的rss地址即可：http://127.0.0.1:8000/all/rss/

###3. Markdown 自动生成文章目录
####3.1 在文中插入目录
	Markdown 在渲染内容的同时还可以自动提取整个内容的目录结构，现在我们来使用 Markdown 为文章自动生成目录。
	在渲染 Markdown 文本时加入了 toc 拓展后，就可以在文中插入目录了
	方法是在书写 Markdown 文本时，在你想生成目录的地方插入 [TOC] 标记即可。
	例如新写一篇 Markdown 博文，其 Markdown 文本内容如下：
		[TOC]
		## 我是标题一 
		这是标题一下的正文 
		## 我是标题二 
		这是标题二下的正文 
		### 我是标题二下的子标题 
		这是标题二下的子标题的正文 
		## 我是标题三 
		这是标题三下的正文
	在[TOC]标记出现的地方就会出现文档的目录。
####3.2 在页面的任何地方插入目录
	上述方式的一个局限局限性就是只能通过 [TOC] 标记在文章内容中插入目录。
	加入了markdown的toc扩展之后，实例化后的markdown对象调用convert方法将会获得toc属性，代表的是内容的目录。
	在PostDetailView中修改get_object方法，如下：
		    def get_object(self, queryset=None):
	        # 复写get_object方法的目的是因为需要对post的body值进行渲染
	        post = super(PostDetailView, self).get_object(queryset=None)
	        md = markdown.Markdown(extensions=[
	            'markdown.extensions.extra',
	            'markdown.extensions.codehilite',
	            'markdown.extensions.toc'
	        ])
	        post.content = md.convert(post.body)
	        post.toc = md.toc
	        return post
	注意：post本身并没有toc这个属性，我们我们给它动态添加了toc属性，这是python动态语言的特性。
	在detail.html中的block toc中添加如下代码：
		{% block toc %}
		    <div class="widget widget-content">
		        <h3 class="widget-title">文章目录</h3>
		        {{ post.toc |safe }}
		    </div>
		{% endblock %}
####3.3 美化标题的锚点 URL
	文章内容的标题被设置了锚点，点击目录中的某个标题，页面就会跳到该文章内容中标题所在的位置。
	这时候浏览器的 URL 显示的值可能不太美观，比如像下面的样子：
		http://127.0.0.1:8000/post/10/#_4
	#_4 就是锚点。
	Markdown 在设置锚点时利用的是标题的值.由于通常我们的标题都是中文，Markdown 没法处理。所以它就忽略的标题的值，而是简单地在后面加了个 _1 这样的锚点值。
	django.utils.text 中的 slugify 方法来处理标题的中文问题。
	from django.utils.text import slugify
	from markdown.extensions.toc import TocExtension	
		def get_object(self, queryset=None):
	        post = super(PostDetailView, self).get_object(queryset=None)
	        md = markdown.Markdown(extensions=[
	            'markdown.extensions.extra',
	            'markdown.extensions.codehilite',
	            TocExtension(slugify=slugify),
	        ])
	        post.body = md.convert(post.body)
	        post.toc = md.toc
	        return post

###4. 简单全文搜索
	搜索是一个复杂的功能，但对于一些简单的搜索任务，我们可以使用 Django Model 层提供的一些内置方法来完成。现在我们来为我们的博客提供一个简单的搜索功能。
####4.0 需求分析
	我们希望用户在搜索框中输入关键词后，能够在标题和正文中查找包含关键词的文章。整个搜索过程如下：
	1. 用户在搜索框中输入搜索关键词，假设为 “django”，然后用户点击了搜索按钮提交其输入的结果到服务器。
	2. 服务器接收到用户输入的搜索关键词 “django” 后去数据库查找文章标题和正文中含有该关键词的全部文章。
	3. 服务器将查询结果返回给用户。
	我们来实现这个过程。
####4.1 将关键词提交给服务器
	修改base.html中导航条的搜索表单的action即可。将用户输入的关键字提交到特定的url。
		<form role="search" method="get" id="searchform" action="{% url 'blog:search' %}">
	        <input type="search" name="q" placeholder="搜索" required>
	        <button type="submit"><span class="ion-ios-search-strong"></span></button>
	    </form>
####4.2 查找含有搜索关键词的文章
	搜索的功能将由 search 视图函数提供：
		def search(request):
		    q = request.GET.get('q')
		    error_msg = ''
		    
		    if not q:
		        error_msg = '请输入关键字'
		        return render(request, 'blog/index.html', {'error_msg': error_msg})
		    post_list = Post.objects.filter(Q(title__icontains=q)|Q(body__icontains=q))
		    return render(request,'blog/index.html',{'error_msg':error_msg,
		                                             'post_list':post_list})
	使用Q对象来进行或查找。
####4.3 渲染搜索结果
	我们复用了 index.html 模板，唯一需要修改的地方就是当有错误信息时，index.html 应该显示错误信息。只需要在文章列表前加个 error_msg 模板变量即可：
		{% block main %}
		    {% if error_msg %}
		        <p>{{ error_msg }}</p>
		    {% endif %}
			...
		{%endblock%}
####4.4 绑定 URL
	有了视图函数后记得把视图函数映射到相应的URL，如下：
	url(r'^search/$', views.search, name='search')

###4. Django Haystack 全文检索与关键词高亮
	我们已经实现的搜索功能实在过于简单，没有多大的实用性。对于一个搜索引擎来说，至少应该能够根据用户的搜索关键词对搜索结果进行排序以及高亮关键字。
####4.0 Django Haystack 简介
	django-haystack 是一个专门提供搜索功能的 django 第三方应用，
	它支持 Solr、Elasticsearch、Whoosh、Xapian 等多种搜索引擎
	配合著名的中文自然语言处理库 jieba 分词，
	就可以为我们的博客提供一个效果不错的博客文章搜索系统。
####4.1 安装必要依赖
	激活虚拟环境执行:
	pip install whoosh django-haystack jieba -i https://pypi.douban.com/simple/
#### 4.2 配置 Haystack
	首先是把 django haystack 加入到 INSTALLED_APPS 选项里.
	然后加入如下配置项：
		HAYSTACK_CONNECTIONS = {
	    'default':{
		        'ENGINE':'blog.whoosh_cn_backend.WhooshEngine',
		        'PATH':os.path.join(BASE_DIR,'whoosh_index'),
		    }
		}
		HAYSTACK_SEARCH_RESULTS_PER_PAGE = 10
		HAYSTACK_SIGNAL_PROCESSOR = 'haystack.signals.RealtimeSignalProcessor'
	参数说明：
		HAYSTACK_CONNECTIONS 的 ENGINE 指定了 django haystack 使用的搜索引擎。
		PATH 指定了索引文件（搜索引擎需要建立索引文件）需要存放的位置。
		HAYSTACK_SEARCH_RESULTS_PER_PAGE 指定如何对搜索结果分页，这里设置为每 10 项结果为一页。
		HAYSTACK_SIGNAL_PROCESSOR 指定什么时候更新索引，这里我们使用 haystack.signals.RealtimeSignalProcessor，作用是每当有文章更新时就更新索引。由于博客文章更新不会太频繁，因此实时更新没有问题。
####4.3 处理数据
	接下来就要告诉 django haystack 使用哪些数据建立索引以及如何存放索引。
	如果要对 blog 应用下的数据进行全文检索，做法是在 blog 应用下建立一个search_indexes.py 文件，写上如下代码：
		from haystack import indexes
		from .models import Post
		
		class PostIndex(indexes.SearchIndex, indexes.Indexable):
		    text = indexes.CharField(document=True, use_template=True)
		
		    def get_model(self):
		        return Post
		
		    def index_queryset(self, using=None):
		        return self.get_model().objects.all()
	之所以写上述代码是因为，haystack规定：要想对某个 app 下的数据进行全文检索，
	就要在该 app 下创建一个 search_indexes.py 文件，
	然后创建一个 XXIndex 类（XX 为含有被检索数据的模型，
	如这里的 Post），并且继承 SearchIndex（注意不是SearchField) 和 Indexable。
	上述代码创建了一个针对Post的索引。为什么创建索引，是因为创建索引可以加快搜索速度减小服务器压力。有点类似数据库索引。
	每个索引里面必须有且只能有一个字段为 document=True，
	这代表 django haystack 和搜索引擎将使用此字段的内容
	作为索引进行检索(primary field)。
	如果使用一个字段设置了document=True，则一般约定此字段名为text
	这是在 SearchIndex 类里面一贯的命名，以防止后台混乱。
	当然名字你也可以随便改，不过不建议改。就和request和response命名一样。
	haystack 提供了use_template=True 在 text 字段中，这样就允许我们使用数据模板去建立搜索引擎索引的文件.
		模板路径为：templates/search/indexes/blog/post_text.txt
		其内容为：
			{{ object.title }} 
			{{ object.content }}
		这个数据模板的作用是对 Post.title、Post.content 
		这两个字段建立索引，当检索的时候会对这两个字段做
		全文检索匹配，然后将匹配的结果排序后作为搜索结果返回。
####4.4 配置 URL
	搜索的视图函数和 URL 模式 django haystack 都已经帮我们写好了，直接include(haystack.urls)即可。
	同时删掉我们之前配的url，以防止冲突。
####4.5 修改搜索表单
	修改一下搜索表单，让它提交数据到 django haystack 搜索视图对应的 URL：	
		<form role="search" method="get" id="searchform" action="{% url 'haystack_search' %}">
####4.6 创建搜索结果页面
	默认情况下，haystack_search 视图函数会将搜索结果传递给模板 search/search.html
	因此创建这个模板文件，对搜索结果进行渲染。
	search.html模板详情见项目目录。
	这个模板基本和 blog/index.html 一样，只是由于 haystack 对搜索结果做了分页，传给模板的变量是一个 page 对象，所以我们从 page 中取出这一页对应的搜索结果，然后对其循环显示，即 {% for result in page.object_list %}。

####4.7 高亮关键词
	haystack 中实现关键词高亮效果非常简单，只需要使用 {% highlight %} 模板标签即可。
	{% highlight result.content with query %}
	高亮处理的原理其实就是给文本中的关键字包上一个 span 标签并且为其添加 highlighted 样式。
	我们还要给 highlighted 类指定样式，在 base.html 中添加即可：
	<style> 
		span.highlighted { color: red; } 
	</style>
####4.8 修改搜索引擎为中文分词
	Whoosh 作为搜索引擎，但在 django haystack 中为 Whoosh 指定的分词器是英文分词器，可能会使得搜索结果不理想。
	我们把这个分词器替换成 jieba 中文分词器。
	从安装的site-packages目录的 haystack 目录中把 haystack/backends/whoosh_backend.py 文件拷贝到 blog/ 下，重命名为 whoosh_cn_backend.py
	然后找到如下一行代码：
	【blog/ whoosh_cn_backend.py】 
	schema_fields[field_class.index_fieldname] = TEXT(stored=True, analyzer=StemmingAnalyzer(), field_boost=field_class.boost, sortable=True)
	将其中的 analyzer 改为 ChineseAnalyzer，当然为了使用它，你需要在文件顶部引入：from jieba.analyse import ChineseAnalyzer。
####4.9 建立索引文件
	最后一步就是建立索引文件了，运行命令 python manage.py rebuild_index 就可以建立索引文件了。

​	
​	
​		
​		
					
​			
​	
​	