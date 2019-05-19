---
date: 2017-02-12 21:20
status: public
title: Django web framework
---

# 0 Preparation
There are a lot of python web frameworks for us to choose, such as `flask`, `Tornado` and `Django`. The reasons for choosing [Django](https://www.djangoproject.com/) are listed below.
+ A lot of start-up commpanies using django in production.
+ Cookbooks for django can be found everywhere by google.
+ I'm a fan of Quentin Tarantino who directed a film called `Django Unchained` in 2012. 

# 1 Django shell interface
## 1.1 start project
`django-admin startproject <projectname>` in terminal, your workspace directionary will get <projecename> folder which contains the same name folder. 
+ `__init__.py`  
The python package signle
+ `setting.py`  
The configuration of this web app, such as `database`, `template`, `static` and so on.
+ `urls.py`  
The whole url mapping system which uses regular expressions.
+ `wsgi.py`  
the web application
Outsides the folder, `manage.py` provides most part shell commands.

## 1.2 create app
`python manage.py startapp <appname>`, django will help you create <appanme> folder. In most cases, a web project holds a lot of applicaiton, so project-folder is the parent of application folder. By doing this, you can reuse each applicaiton in different projects. 

## 1.3 database migrate
Projection's models are stored in database. Django uses **ORM** mechanism connect model and table. Once you change schame of model in `appname/models.py`, Using this two commands.
+ `python manage.py makemigrations appname`  
print the infomation to migrate.
+ `python manage.puy migrate`   
migrate the model to database's tables. 
## 1.4 run web applicaiton 
`python manmage.py runserver <port>` open browser and open `127.0.0.1:<port>`.   

# 2 url mapping
A typical `urls.py` is posted below.
```python
from django.conf.urls import url
from . import views
urlpatterns = [
    url(r'^$', views.index, name='index'),
    url(r'^about/$', views.about, name='about'),
    url(r'^category/(?P<category_name_slug>[\w-]+)/$', views.show_category, name='show_category'),
    url(r'^add_category/$', views.add_category, name='add_category'),
    url(r'^category/(?P<category_name_slug>[\w\-]+)/add_page/$', views.add_page, name='add_page'), 
]
```
According regular expression rule `^` matches the begining of a string and `$` matches the end of a string. So `^$` mathes the `127.0.0.1:8000/` , `^about/$` matches the `127.0.0.1:8000/about/` and so on.    
`(?P<category_name_slug>[\w-]+)` called named regular expression. `[\w-]+` matches letter and `-`, and named it `category_name_slug`. which can be the parameter in `show_category` function in views module. Above all, the django can create elegant and simple urls. 
url can map to different application. in projectionname folder, a urls.py looks like that, 
```python
from django.conf.urls import url, include
from django.contrib import admin
from rango import views
urlpatterns = [
    url(r'^$', views.index, name='index'),
    url(r'^rango/', include('rango.urls')),
    url(r'^admin/', admin.site.urls),
]
```
the second url pattern uses `include` mapping '127.0.0.1:8000/rango/...' to `rango` application's `urls.py` pattern. 

# 3 Model-Template-View
When urls module determined which view reponses the http request, the view return a html page or json. 
## 3.1 simple html
Once http `get` request likes `127.0.0.1:800/rango/about/` reaches, `about` function will return a wrapper html text to client. 
```python
def about(request):
    return HttpResponse(r'<b> about page </b> </br> <a href="/rango/">Index</a>')
```
## 3.2 parameter request
In url module regular expression, pass parameter to function.
```python
def show_category(request, category_name_slug):
    context_dict = {}
    try:
        category = models.Category.objects.get(slug=category_name_slug)
        pages = models.Page.objects.filter(category=category)
        context_dict['pages'] = pages
        context_dict['category'] = category
    except models.Category.DoesNotExist:
        context_dict['pages'] = None
        context_dict['category'] = None
    return render(request, 'rango/category.html', context_dict)
```
In this case, the `category_name_slug` is matched by regular expression, which represents the `category name`. Fetching data from database where entry's slug equals category_name_slug, the result stored in dictionary and pass to template. 
## 3.3 template
Template is the result of web application. Unlike static web page, dynamic web application should reponse the client request. So you should fill data into html template.
```html
{% load staticfiles %} 
<html>
    <head>
        <title>Rango</title>
    </head>    
    <body>
        <h1>Rango says...</h1>
        <div>hey there parenter!</div>        
        <div>
            {% if categories %}
            <ul>
                {% for category in categories %}
                    <li>
                    <a href="/rango/category/{{ category.slug }}">
                    {{ category.name }}</a>
                    </li>
                {% endfor %}
            </ul>
            {% else %}
                <strong>There are no category present!</strong>
            {% endif %}
        </div>
        <div>
            <a href="/rango/add_category/">Add a New Category</a><br />
        </div>        
        <div>
            <a href="/rango/about/">About</a><br />
            <img src="{% static "images/media.jpg" %}"
                 alt="Picture of Rango" /> <!-- New line -->
        </div>
    </body>    
</html>
```
This is skeleton of html file which is filled `{% %}`. The sentences are same with python language and this are the key form dictionary in views module. 
# 4 Form
When you want to post some infomation to server, you should fill a form in  browser and submit it.
## 4.1 form model
Django provide the `ModelForm` base class for users. You just inherit it and add some fields asking user filling. 
```python
class CategoryForm(forms.ModelForm):
    name = forms.CharField(max_length=128, 
                        help_text="Please enter the category name.")
    views = forms.IntegerField(widget=forms.HiddenInput(), initial=0)
    likes = forms.IntegerField(widget=forms.HiddenInput(), initial=0)
    slug = forms.CharField(widget=forms.HiddenInput(), required=False)
    # An inline class to provide additional information on the form.
    class Meta:
        # Provide an association between the ModelForm and a model
        model = Category
        fields = ('name',)
```
## 4.2 views 
Simply, every http request can be divided into `post` and `get`, which share the same url. So, the views' function will be different when facing form situation.
```python
def add_category(request):
    form = CategoryForm()
    if request.method == 'POST':
        form = CategoryForm(request.POST)
        if form.is_valid():
            form.save(commit=True)
            return index(request)
        else:
            print(form.errors)
    else:
        return render(request, 'rango/add_category.html', {'form' : form})
```
If the request is `POST`, save the infomation user submitted. and return the html page which contains the form.
## 4.3 html
```html
<html>
    <head>
        <title>Rango</title>
    </head>
    <body>
        <h1>Add Category</h1>
        <div>
            <form id='category_form' method='POST'>
                {% csrf_token %}
                {% for hidden in form.hidden_fields %}
                    {{ hidden }}
                {% endfor %}
                {% for field in form.visible_fields %}
                    {{ field.errors }}
                    {{ field.help_text }}
                    {{ field }}
                {% endfor %}
                <input type="submit" name="submit" 
                value="Create Category" />
            </form>
        </div>
    </body>
</html>
```