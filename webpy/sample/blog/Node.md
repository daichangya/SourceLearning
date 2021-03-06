# web.py 项目之 blog

## 目录树

```
src/
├── blog.py
├── model.py
├── schema.sql
└── templates
    ├── base.html
    ├── edit.html
    ├── index.html
    ├── new.html
    └── view.html

1 directory, 8 files
```

## 项目说明

项目来自 [webpy.org](http://webpy.org/src/blog/0.3),
主要实现一个基于web的博客系统，实现了基本的增删查改。

## 结构分析

### 控制模块

控制模块包括 blog.py

```python
""" Basic blog using webpy 0.3 """
import web
import model

### Url mappings

urls = (
    '/', 'Index',
    '/view/(\d+)', 'View',
    '/new', 'New',
    '/delete/(\d+)', 'Delete',
    '/edit/(\d+)', 'Edit',
)


### Templates
t_globals = {
    'datestr': web.datestr
}
render = web.template.render('templates', base='base', globals=t_globals)


class Index:

    def GET(self):
        """ Show page """
        posts = model.get_posts()
        return render.index(posts)


class View:

    def GET(self, id):
        """ View single post """
        post = model.get_post(int(id))
        return render.view(post)


class New:

    form = web.form.Form(
        web.form.Textbox('title', web.form.notnull, 
            size=30,
            description="Post title:"),
        web.form.Textarea('content', web.form.notnull, 
            rows=30, cols=80,
            description="Post content:"),
        web.form.Button('Post entry'),
    )

    def GET(self):
        form = self.form()
        return render.new(form)

    def POST(self):
        form = self.form()
        if not form.validates():
            return render.new(form)
        model.new_post(form.d.title, form.d.content)
        raise web.seeother('/')


class Delete:

    def POST(self, id):
        model.del_post(int(id))
        raise web.seeother('/')


class Edit:

    def GET(self, id):
        post = model.get_post(int(id))
        form = New.form()
        form.fill(post)
        return render.edit(post, form)


    def POST(self, id):
        form = New.form()
        post = model.get_post(int(id))
        if not form.validates():
            return render.edit(post, form)
        model.update_post(int(id), form.d.title, form.d.content)
        raise web.seeother('/')


app = web.application(urls, globals())

if __name__ == '__main__':
    app.run()
```

可以看出，主要的逻辑，即对各个请求的处理，都在这个模块中，
包括表单的生成，数据的获取，以及对模板引擎的调用。这个
模块主要是在顶层调用，具体的细节不在这里实现。


### 显示模块

显示模块没有具体实现逻辑的代码，只有模板引擎用到的一些HTML
文件，这些文件的大体内容已经定下来，只是具体需要填充的内容
需要在控制模块中进行设定。


templates/base.html  

```html
$def with (page)

<html>
<head>
    <title>My Blog</title>
    <style>
        #menu {
            width: 200px;
            float: right;
        }
    </style>
</head>
<body>

<ul id="menu">
    <li><a href="/">Home</a></li>
    <li><a href="/new">New Post</a></li>
</ul>

$:page

</body>
</html>
```

`base.html` 为每个文件提供都基本的模板，其它文件只负责自己的那
一块内容。


templates/index.html

```html

$def with (posts)

<h1>Blog posts</h1>

<ul>
$for post in posts:
    <li>
        <a href="/view/$post.id">$post.title</a> 
        from $datestr(post.posted_on) 
        <a href="/edit/$post.id">Edit</a>
    </li>
</ul>
```

`index.html` 负责文章列表的显示。

templates/new.html  

```html

$def with (form)


<h1>New Blog Post</h1>
<form action="" method="post">
$:form.render()
</form>
```

`new.html` 负责新建文件页面。

templates/edit.html

```html

$def with (post, form)

<h1>Edit $form.d.title</h1>

<form action="" method="post">
$:form.render()
</form>


<h2>Delete post</h2>
<form action="/delete/$post.id" method="post">
    <input type="submit" value="Delete post"/>
</form>
```

`edit.html` 负责文章的编辑页面。

templates/view.html

```html
$def with (post)

<h1>$post.title</h1>
$datestr(post.posted_on)<br/>

$post.content
```

`view.html` 负责显示文章页面。


### 数据模块

数据模块包含两个文件　`schema.sql` 和 `model.py`

schema.sql  

```
CREATE TABLE entries (
    id INT AUTO_INCREMENT,
    title TEXT,
    content TEXT,
    posted_on DATETIME,
    primary key (id)
);
```

`schema.sql` 包含了数据库中表的格式

model.py

```python

import web, datetime

db = web.database(dbn='mysql', db='miniblog', user='root', pw="7102155")

def get_posts():
    return db.select('entries', order='id DESC')

def get_post(id):
    try:
        return db.select('entries', where='id=$id', vars=locals())[0]
    except IndexError:
        return None

def new_post(title, text):
    db.insert('entries', title=title, content=text, posted_on=datetime.datetime.utcnow())

def del_post(id):
    db.delete('entries', where="id=$id", vars=locals())

def update_post(id, title, text):
    db.update('entries', where="id=$id", vars=locals(),
        title=title, content=text)
```

`model.py` 包含了数据库连接的创建与数据库操作的具体细节。
此模块中的函数被控制模块调用，从数据库中读取数据。


总的来说，控制模块在最顶层，负责对请求进行处理，调用数据模块，
从数据库中读取，写入数据。再将从数据库中取出的数据送给显示模块
进行显示。

感觉项目再大一点就应该把控制模块再分割了。
