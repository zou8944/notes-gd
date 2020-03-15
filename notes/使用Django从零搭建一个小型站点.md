# 使用Django从零搭建一个小型站点

## 一. 要有project

创建一个基本的Project, 这样我们才能开始工作

```shell
django-admin startproject <工程名>
```

## 二. 要能访问数据库

Django说, 要访问数据库必须创建app和model, 于是, 我们开始创建

- 首先配置好数据库

  在settings.py中加上自己的数据库配置

  ```python
  DATABASES = {   
      'default': {        
          'ENGINE': 'django.db.backends.sqlite3',       
          'NAME': os.path.join(BASE_DIR, 'db.sqlite3'),    
      },   
      # 如下是我的配置
      'test': {      
          'ENGINE': 'django.db.backends.mysql',      
          'NAME': 'test',      
          'USER': 'test',        
          'PASSWORD': 'test123',     
          'HOST':'localhost',      
          'PORT':'3306',   
      }
  }
  ```

- 然后定义好模型

  在manager.py文件同目录下, 执行如下命令创建app

  ```shell
  django-admin startapp KnowledgePayModel
  ```

  然后可以看到如下目录结构

  ![1566014155470](/home/floyd/.config/Typora/typora-user-images/1566014155470.png)

  然后在project的settings.py下注册刚刚创建的APP

  ```python
  INSTALLED_APPS = [
      'django.contrib.admin',
      'django.contrib.auth',
      'django.contrib.contenttypes',
      'django.contrib.sessions',
      'django.contrib.messages',
      'django.contrib.staticfiles',
      # 这是我们自己创建的APP
      'KnowledgePayModel',
  ]
  ```

  然后执行如下命令从现有数据表中导出模型

  ```shell
  python manager.py inspectdb 表名
  ```

  我使用django + psycopg2 2.8.3执行上述命令时出现如下错误

  ```python
  from django.db import models
  # Unable to inspect table 'payalbum'
  # The error was: sequence index must be integer, not 'slice'
  # Unable to inspect table 'paycomment'
  # The error was: sequence index must be integer, not 'slice'
  # Unable to inspect table 'payprice'
  # The error was: sequence index must be integer, not 'slice'
  # Unable to inspect table 'payvideo'
  # The error was: sequence index must be integer, not 'slice'
  ```

  网上查询是版本兼容有问题, 于是将psycopg强制降级

  ```shell
  sudo pip install psycopg2==2.7.7 --force-reinstall
  ```

  然后就可以正常工作了, 结果如下, 将其复制到models.py中, 然后进行适当的修改即可使用. 注意生成的AutoField一定要指定为主键, 否则Django会认为没有主键, 从在再隐式给你加一个, 然后就会报说一个model中只能有一个AutoField域.

  ```python
  # This is an auto-generated Django model module.
  # You'll have to do the following manually to clean this up:
  #   * Rearrange models' order
  #   * Make sure each model has one field with primary_key=True
  #   * Make sure each ForeignKey has `on_delete` set to the desired behavior.
  #   * Remove `managed = False` lines if you wish to allow Django to create, modify, and delete the table
  # Feel free to rename the models, but don't rename db_table values or field names.
  from __future__ import unicode_literals
  
  from django.db import models
  
  class Payalbum(models.Model):
      id = models.AutoField()
      data = models.TextField()  # This field type is a guess.
  
      class Meta:
          managed = False
          db_table = 'payalbum'
  
  class Paycomment(models.Model):
      id = models.AutoField()
      data = models.TextField()  # This field type is a guess.
  
      class Meta:
          managed = False
          db_table = 'paycomment'
  
  class Payprice(models.Model):
      id = models.AutoField()
      data = models.TextField()  # This field type is a guess.
  
      class Meta:
          managed = False
          db_table = 'payprice'
  
  class Payvideo(models.Model):
      id = models.AutoField()
      data = models.TextField()  # This field type is a guess.
  
      class Meta:
          managed = False
          db_table = 'payvideo'
  ```

  如果想要使用Django自带的权限表之类的, 需要执行如下语句, 来生成一系列表

  ```shell
  python manager.py migrate
  ```
  
  控制台执行结果如下:
  
  ```shell
  Operations to perform:
    Apply all migrations: admin, auth, contenttypes, sessions
  Running migrations:
    Applying contenttypes.0001_initial... OK
    Applying auth.0001_initial... OK
    Applying admin.0001_initial... OK
    Applying admin.0002_logentry_remove_auto_add... OK
    Applying contenttypes.0002_remove_content_type_name... OK
    Applying auth.0002_alter_permission_name_max_length... OK
    Applying auth.0003_alter_user_email_max_length... OK
    Applying auth.0004_alter_user_username_opts... OK
    Applying auth.0005_alter_user_last_login_null... OK
    Applying auth.0006_require_contenttypes_0002... OK
    Applying auth.0007_alter_validators_add_error_messages... OK
    Applying auth.0008_alter_user_username_max_length... OK
    Applying sessions.0001_initial... OK
  
  ```
  
  数据库里面如下
  
  ![1566017457779](/home/floyd/.config/Typora/typora-user-images/1566017457779.png)

## 三. 要有静态资源

将所有静态资源全部放入templates, 得到的文件结构如下

![1566016251249](/home/floyd/.config/Typora/typora-user-images/1566016251249.png)

修改settings.py中的静态文件位置配置

```python
STATIC_URL = '/templates/'
STATICFILES_DIRS = (
    os.path.join(BASE_DIR, "templates"),
)
```

在TEMPLATES的DIRS项中增加templates文件夹

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        # 只有这个templates是我添加的哦
        'DIRS': ['templates'],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```

## 四. 要有路由

在project的urls.py文件中添加我们的路由

```python
urlpatterns = [
    url(r'^admin/', admin.site.urls),
    # 将knowledgepay这个url导向server这个app,让它自己处理
    url(r'^knowledgepay/', include('server.urls', namespace="server")),
]
```

在我们自己的app中创建一个urls.py, 定义app内部自己的路由

```python
# 这是app内部的路由
urlpatterns = [
    url(r'^$', views.homepage),
    url(r'^home', views.homepage),
]
```

在我们自己app的views.py文件中定义各种视图处理器

```python
# Create your views here.
def homepage(request):
    return render_to_response('login.html')
```

## 五. 发挥吧

上面已经把基本的框架都搭好了.我们已经可以

- 可以对url进行路由
- 可以访问静态html文件
- 可以访问数据库
- 可以访问自定义的逻辑处理方法

剩下的就是纯粹业务上的事情了.

## 六. 参考资料

如果觉得上面有些地方前后没有联系起来, 那么可以结合下面这个简单的菜鸟教程看看

https://www.runoob.com/django/django-tutorial.html

要详细了解, 这里还有Djang中文版文档可供参阅

https://docs.djangoproject.com/zh-hans/2.2/intro/

当然这里还有Django book的中文版翻译, 供君选择

http://docs.30c.org/djangobook2/index.html