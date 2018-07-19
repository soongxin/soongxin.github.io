---
layout: post
title: 教程4-身份验证&权限
date: 2016-01-01
categories: test
tags: Python
---
现在我们的视图对谁可以编辑或者删除我们的代码片段没有做任何限制，我们需要一些更高级的功能以确保如下几点：
* 将代码片段与创造者联系起来。
* 只有通过身份验证的用户才可以创建代码片段。
* 只有创建者可以修改或者删除代码片段。
* 未通过认证的用户应有完整的只读权限。
## 向模型类中添加信息
我们将对我们的`Snippet`模型类做一些修改，首先，添加一些字段。其中有一个字段用来表示代码片段的坐这，其他字段用来存储在HTML中高亮显示的规则。
将这两个字段添加到`models.py`文件中的`Snippet`类中。
```python
owner = models.ForeignKey('auth.User', related_name='snippets', on_delete=models.CASCADE)
highlighted = models.TextField()
```
我们还需要确保模型类被保存时，通过`pygments`代码高亮库填充高亮字段。
这需要一些额外的导入：
```python
from pygments.lexers import get_lexer_by_name
from pygments.formatters.html import HtmlFormatter
from pygments import highlight
```
现在可以为我们的模型类添加`.save()`方法了：
```python
def save(self, *args, **kwargs):
    """
    Use the `pygments` library to create a highlighted HTML
    representation of the code snippet.
    """
    lexer = get_lexer_by_name(self.language)
    linenos = 'table' if self.linenos else False
    options = {'title': self.title} if self.title else {}
    formatter = HtmlFormatter(style=self.style, linenos=linenos,
                              full=True, **options)
    self.highlighted = highlight(self.code, lexer, formatter)
    super(Snippet, self).save(*args, **kwargs)
```
准备工作已经完成，我们需要更新数据库表，通常我们创建迁移脚本来更新表，但是根据本教程的实际情况，我们直接删掉数据库重新开始（请不要再生产环境下这样做）：
```python
rm -f db.sqlite3
rm -r snippets/migrations
python manage.py makemigrations snippets
python manage.py migrate
```
你可能想要创建几个不同的用户来测试API，最快的方法是使用`createsuperuser`命令。
```python
python manage.py createsuperuser
```
## User模型类的收尾工作
现在我们有了几个可以工作的用户，我们最为这些用户在我们的API中添加表示方法，创建一个新的序列化器很容易，在`serializers.py`中添加：
```python
from django.contrib.auth.models import User
class UserSerializer(serializers.ModelSerializer):
    snippets = serializers.PrimaryKeyRelatedField(many=True, queryset=Snippet.objects.all())
    class Meta:
        model = User
        fields = ('id', 'username', 'snippets')
```
由于`snippets`是User模型类的反向引用的的字段，当我们使用`ModelSerializer`的时候，它不会被默认包含进去，所以我们需要明确指定该字段。
我们仍需在`views.py`中添加一些视图，我们只想给用户表现添加只读视图，所以我们使用基于generic类视图的`ListAPIViews`和`RetrieveAPIView`。
```python
from django.contrib.auth.models import User
class UserList(generics.ListAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
class UserDetail(generics.RetrieveAPIView):
    queryset = User.objects.all()
    serializer_class = UserSerializer
```
确认是否导入了`UserSerializer`了：
```python
from snippets.serializers import UserSerializer
```
最后我们需要配置URL，把下面的内容添加到`urls.py`中
```python
url(r'^users/$', views.UserList.as_view()),
url(r'^users/(?P<pk>[0-9]+)/$', views.UserDetail.as_view()),
```
## 将代码片段与用户联系起来
现在，如果我们创建代码片段，并没有指定方法将创建代码的用户与代码片段实例联系起来，用户并没有向序列化器添加信息，但是包含在了`request`的属性当中。
解决方法是在我们的`snippet`视图中重写`.perform_create()`方法， 它允许我们修改实例保存的管理方式，并处理传入请求的URL中隐含的任何信息。
在`SnippetList`类视图中，添加如下方法：
```python
def perform_create(self, serializer):
	serializer.save(owner=self.request.user)
```
现在，`create()`方法将传递一个额外的”owner”字段，以及来自请求的验证数据。
## 更新序列化器
现在代码片段与他们的创建者联系起来了，我们修改一下`SnippetSerializer`来展示出来，在`serializers.py`中添加如下字段：
```python
owner = serializers.ReadOnlyField(source='owner.username')
```
> Note: 确保你将`'owner'`字段添加到了`Meta`类中的字段列表里。  
这个字段做了一些十分有趣的事情，`source`参数控制着将要填充到字段中的属性，可以指向序列化器实例的任意属性，它也可以使用上面表示的点标记，在这种情况下，他将以与Django模板语言一样的方式遍历给定的属性。

