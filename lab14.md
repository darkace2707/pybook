# Лабораторная работа №14. Eleven Note

<div class="alert alert-info">
Лабораторная работа основана на <a href="https://github.com/sixfeetup/ElevenNote">замечательном руководстве</a> по Django от компании Six Feet Up.
</div>

Как обычно создайте новую ветку разработки и виртуальное окружение с именем `elevennote-env`, в котором установите следующие пакеты:

```bash
$ pip install Django==1.11.4
$ pip install psycopg2==2.7.3
$ pip install python-decouple==3.1
```

Теперь создадим новый проект с помощью команды `django-admin`. Обратите внимание, что имя проекта `config`, а все файлы проекта будут созданы в текущей рабочей директории (на что указывает `.`):

```bash
$ mkdir elevennote && cd $_
$ django-admin startproject config .
$ ls
config manage.py
```

В результате вы должны получить следующую структуру проекта:
```bash
.
└── elevennote
    ├── config
    │   ├── __init__.py
    │   ├── settings.py
    │   ├── urls.py
    │   └── wsgi.py
    └── manage.py
```

```bash
$ mkdir config/settings
$ mv config/settings.py config/settings/base.py
```

```bash
$ touch config/settings/__init__.py
$ touch config/settings/local.py
$ touch config/settings/production.py
$ touch config/settings/settings.ini
$ tree config
config/
├── __init__.py
├── settings
│   ├── __init__.py
│   ├── base.py
│   ├── local.py
│   ├── production.py
│   └── settings.ini
├── urls.py
└── wsgi.py
```

`config/settings/base.py`
```python
import os
from decouple import config


def root(*dirs):
    base_dir = os.path.join(os.path.dirname(__file__), '..', '..')
    return os.path.abspath(os.path.join(base_dir, *dirs))


BASE_DIR = root()

SECRET_KEY = config('SECRET_KEY')

INSTALLED_APPS = [
    # ...
]

MIDDLEWARE = [
    # ...
]

ROOT_URLCONF = 'config.urls'

TEMPLATES = [
    # ...
]

WSGI_APPLICATION = 'config.wsgi.application'

AUTH_PASSWORD_VALIDATORS = [
    # ...
]

LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'UTC'
USE_I18N = True
USE_L10N = True
USE_TZ = True
```

`configs/settings/local.py`
```python
from .base import *

DEBUG = True

INSTALLED_APPS += [
    'django.contrib.postgres',
]

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': 'localhost',
        'PORT': '',
    }
}
```

В `config/settings/settings.ini` укажите секретный ключ для Django, имя БД, пользователя БД и его пароль:
```python
[settings]
SECRET_KEY=
DB_NAME=
DB_USER=
DB_PASSWORD=
```

<div class="alert alert-warning">
<b>Замечание</b>: Не добавляйте <tt>settings.ini</tt> в ваш репозиторий, так как он содержит пароли и ключи. Чтобы случайно его не закоммитить добавьте соответствующую запись в <tt>.gitignore</tt>.
</div>

`manage.py`
```python
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "config.settings.local")
```

```bash
$ python manage.py migrate
$ python manage.py runserver
```

```bash
$ python manage.py createsuperuser
```

Создадим новое приложение `notes` (заметки) и нашу первую модель `Note`:

```bash
$ python manage.py startapp notes
```

Каждая заметка будет представлена следующими полями:
- заголовком (`title`) с ограничением 200 символов;
- текстом заметки (`body`);
- и датой создания (`pub_date`).

`notes/models.py`
```python
class Note(models.Model):
    title = models.CharField(max_length=200)
    body = models.TextField()
    pub_date = models.DateTimeField('date published')
```

Зарегистрируем созданную модель в панели администратора:
`notes/admin.py`
```python
from django.contrib import admin
from .models import Note

admin.site.register(Note)
```

И применим внесенные изменения к БД:
```bash
$ python manage.py makemigrations notes
$ python manage.py migrate
```

Запустите проект `python manage.py runserver` и откройте панель администратора [http://localhost:8000/admin](http://localhost:8000/admin). Вы должны увидеть, что появился новый раздел `Notes`, в котором вы можете создавать, редактировать и удалять заметки.

![](/assets/Screen Shot 2017-09-16 at 18.10.05.png)

```bash
$ pip install django-admin-honeypot==1.0.0
```

`config/settings/base.py`
```python
INSTALLED_APPS = [
    # ...
    'admin_honeypot',
    # ...
]
```

`config/urls.py`
```python
urlpatterns = [
    url(r'^admin/', include('admin_honeypot.urls', namespace='admin_honeypot')),
    url(r'^secret/', admin.site.urls),
]
```


```bash
$ mkdir requirements
$ pip freeze > requirements/base.txt
```

`notes/admin.py`
```python
class NoteAdmin(admin.ModelAdmin):
    list_display = ('title', 'pub_date', 'was_published_recently')
    list_filter = ['pub_date']

# Replace your other register call with this line:
admin.site.register(Note, NoteAdmin)
```

`notes/models.py`
```python
def was_published_recently(self):
    return self.pub_date >= timezone.now() - timedelta(days=1)
```

![](/assets/Screen Shot 2017-09-16 at 16.31.29.png)

```bash
$ mkdir notes/tests
$ mv notes/tests.py notes/tests/test_models.py
```

`notes/tests/test_models.py`
```python
import datetime

from django.utils import timezone
from django.test import TestCase

from .models import Note

class NoteMethodTests(TestCase):

    def test_was_published_recently(self):
    """
    was_published_recently() should return False for notes whose pub_date is in the future.
    """
        time = timezone.now() + datetime.timedelta(days=30)
        future_note = Note(pub_date=time)
        self.assertEqual(future_note.was_published_recently(), False)
```

```bash
$ python manage.py test
Creating test database for alias 'default'...
F
======================================================================
FAIL: test_was_published_recently (note.tests.NoteMethodTests)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/home/vagrant/Projects/elevennote/elevennote/note/tests.py", line 18, in test_was_published_recently
     self.assertEqual(future_note.was_published_recently(), False)
AssertionError: True != False

----------------------------------------------------------------------
Ran 1 test in 0.001s

FAILED (failures=1)
Destroying test database for alias 'default'...
```

`notes/models.py`
```python
def was_published_recently(self):
    now = timezone.now()
    return now - timedelta(days=1) <= self.pub_date <= now
```

`config/urls.py`
```python
url(r'^notes/', include('notes.urls', namespace="notes")),
```

```bash
$ touch notes/urls.py
```

`notes/urls.py`
```python
from django.conf.urls import url
 
from . import views

urlpatterns = [
    url(r'^$', views.index, name='index'),
    url(r'^(?P<note_id>[0-9]+)/$', views.detail, name='detail'),
]
```

`notes/views.py`
```python
from django.shortcuts import get_object_or_404, render
from django.http import HttpResponse

from .models import Note


def index(request):
    latest_note_list = Note.objects.order_by('-pub_date')[:5]
    context = {
        'latest_note_list': latest_note_list,
    }
    return render(request, 'notes/index.html', context)

def detail(request, note_id):
     note = get_object_or_404(Note, pk=note_id)
     return render(request, 'notes/detail.html', {'note': note})
```

```bash
$ mkdir -p templates/notes
```

`config/settings/base.py`
```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [root('templates')],
        # ...
    },
]
```

`templates/note/index.html`
```html
{% if latest_note_list %}
   <ul>
   {% for note in latest_note_list %}
     <li><a href="{% url 'notes:detail' note.id %}">{{ note.title }}</a></li>
   {% endfor %}
   </ul>
{% else %}
  <p>No notes are available.</p>
{% endif %}
```

`templates/note/detail.html`
```html
<h1>{{ note.title }}</h1>
<p>{{ note.body }}</p>
```

`notes/tests/tests_views.py`
```python

```

```bash
$ python manage.py startapp accounts
$ touch accounts/urls.py
```

`config/urls.py`
```python
#...
from django.http import HttpResponseRedirect

urlpatterns = [
    # Handle the root url.
    url(r'^$', lambda r: HttpResponseRedirect('notes/')),

    # Admin
    url(r'^admin/', include('admin_honeypot.urls', namespace='admin_honeypot')),
    url(r'^secret/', admin.site.urls),

    # Accounts app
    url(r'^accounts/', include('accounts.urls', namespace="accounts")),
 
    # Notes app
    url(r'^notes/', include('note.urls', namespace="notes")),
]
```

```bash
$ touch accounts/urls.py
```

`accounts/urls.py`
```python
from django.conf.urls import url
from django.contrib.auth import views

urlpatterns = [
    url(r'^login/$', views.login, name='login'),
    url(r'^logout/$', views.logout, name='logout'),
]
```

`templates/registration/login.html`
```html
<form action="{% url 'accounts:login' %}" method="post" accept-charset="utf-8">
  {% csrf_token %}
  {% for field in form %}
    <label>{{ field.label }}</label>
    {% if field.errors %}
      {{ field.errors }}
    {% endif %}
    {{ field }}
  {% endfor %}
  <input type="hidden" name="next" value="{{ next }}" />
  <input class="button small" type="submit" value="Submit"/>
</form>
```

`notes/views.py`
```python
# ...
from django.contrib.auth.decorators import login_required

@login_required
def index(request):
    # ...


@login_required
def detail(request):
    # ...
```

`notes/tests/test_views.py`
```python

```


```bash
$ mkdir static
$ cd static
$ wget https://github.com/twbs/bootstrap/releases/download/v4.0.0-alpha.6/bootstrap-4.0.0-alpha.6-dist.zip
$ unzip bootstrap-4.0.0-alpha.6-dist.zip
$ mv bootstrap-4.0.0-alpha.6-dist bootstrap
$ cd ..
```

`config/settings/base.py`
```python
# ...
STATIC_URL = '/static/'
STATICFILES_DIRS = [
    root('static'),
]
```

```bash
# Fix me
wget https://github.com/sixfeetup/ElevenNote/raw/master/templates-ch06.zip
unzip templates-ch6.zip   # If prompted to replace, say (Y)es
cd ..
```


`accounts/views.py`
```python
from django.contrib.auth import authenticate, login
from django.contrib.auth.forms import UserCreationForm
from django.views.generic import FormView


class RegisterView(FormView):
    template_name = 'registration/register.html'
    form_class = UserCreationForm
    success_url='/'
    
    def form_valid(self, form):
        #save the new user first
        form.save()
        
        #get the username and password
        username = self.request.POST['username']
        password = self.request.POST['password1']
        
        #authenticate user then login
        user = authenticate(username=username, password=password)
        login(self.request, user)
        return super(RegisterView, self).form_valid(form)
```

`accounts/urls.py`
```python
from django.conf.urls import url
from django.contrib.auth import views as auth_views
from django.core.urlresolvers import reverse_lazy

from .views import RegisterView

urlpatterns = [
    url(r'^login/$', auth_views.login, name='login'),
    url(r'^logout/$', auth_views.logout, {"next_page" : reverse_lazy('accounts:login')}, name='logout'),
    url('^register/', RegisterView.as_view(), name='register'),
]
```


`config/settings/base.py`
```python
# ...
LOGIN_REDIRECT_URL = '/'
```

`accounts/tests/test_views.py`
```python

```

`notes/models.py`
```python
# ...
from django.contrib.auth.models import User


class Note(models.Model):
    # ...
    owner = models.ForeignKey(User, related_name='notes', on_delete=models.CASCADE)
```

```bash
$ python manage.py makemigrations
$ python manage.py migrate
```

`notes/views.py`
```python
latest_note_list = Note.objects.filter(owner=request.user).order_by('-pub_date')[:5]
```

`notes/admin.py`
```python
list_display = ('title', 'owner', 'pub_date', 'was_published_recently')
```


`notes/views.py`
```python
from django.utils.decorators import method_decorator
from django.contrib.auth.decorators import login_required
from django.views.generic import ListView, DetailView

from .models import Note


class NoteList(ListView):
    paginate_by = 5
    template_name = 'notes/index.html'
    context_object_name = 'latest_note_list'

    @method_decorator(login_required)
    def dispatch(self, *args, **kwargs):
        return super(NoteList, self).dispatch(*args, **kwargs)

    def get_queryset(self):
        return Note.objects.filter(owner=self.request.user)


class NoteDetail(DetailView):
    model = Note
    template_name = 'notes/detail.html'
    context_object_name = 'note'

    @method_decorator(login_required)
        def dispatch(self, *args, **kwargs):
            return super(NoteDetail, self).dispatch(*args, **kwargs)
```

`notes/urls.py`
```python
from django.conf.urls import url

from .views import NoteList, NoteDetail

urlpatterns = [
    url(r'^$', NoteList.as_view(), name='index'),
    url(r'^(?P<pk>[0-9]+)/$', NoteDetail.as_view(), name='detail'),
]
```

`notes/views.py`
```python
from django.contrib.auth.mixins import LoginRequiredMixin


class NoteList(LoginRequiredMixin, ListView):
    # ...

class NoteDetail(LoginRequiredMixin, DetailView):
    # ...
```

<div class="alert alert-info">
В <a href="https://github.com/sixfeetup/ElevenNote/wiki/09-Intro-to-Mixins">руководстве</a> от Six Feet Up объясняется как создать собственный миксин, который бы решал аналогичную задачу.
</div>

`notes/forms.py`

```python
from django import forms

from .models import Note

class NoteForm(forms.ModelForm):

    class Meta:
        model = Note
        exclude = ['owner', 'pub_date']
```

`notes/views.py`
```python
from django.views.generic import ListView, DetailView, CreateView
from django.utils import timezone
from django.core.urlresolvers import reverse_lazy

from .forms import NoteForm
...

class NoteCreate(LoginRequiredMixin, CreateView):
    form_class = NoteForm
    template_name = 'notes/form.html'
    success_url = reverse_lazy('notes:index')

    def form_valid(self, form):
        form.instance.owner = self.request.user
        form.instance.pub_date = timezone.now()
        return super(NoteCreate, self).form_valid(form)
```

`notes/urls.py`
```python
url(r'^new/$', NoteCreate.as_view(), name='create'),
```

`templates/notes/form.html`
```html
{% extends "base.html" %}

{% block content %}

{% if form.errors %}
    {% for error in form.non_field_errors %}
        <div class="alert alert-danger" role="alert">
            <strong>{{ error|escape }}</strong>
        </div>
    {% endfor %}
{% endif %}

<form action="{% url 'notes:create' %}" method="post" accept-charset="utf-8">
    {% csrf_token %}
    {% for field in form %}
    <p>
        <label>{{ field.label }}</label>
        {% if field.errors %}
	<div class="alert alert-danger" role="alert">
            {{ field.errors }}
	</div>
        {% endif %}
        {{ field }}
    </p>
    {% endfor %}
    <input type="hidden" name="next" value="{{ next }}" />
    <input class="button small" type="submit" value="Submit"/>
</form>

{% endblock %}
```

`templates/notes/index.html`
```html
<a href="{% url 'notes:create' %}">Create a new note</a>
```

`template/notes/index.html`
```html
{% if is_paginated %}
<div class="pagination">
   <span class="step-links">
       {% if page_obj.has_previous %}
           <a href="?page={{ page_obj.previous_page_number }}">previous</a>
       {% endif %}

       <span class="current">
           Page {{ page_obj.number }} of {{ page_obj.paginator.num_pages }}.
       </span>

       {% if page_obj.has_next %}
          <a href="?page={{ page_obj.next_page_number }}">next</a>
       {% endif %}
  </span>
</div>
{% endif %}
```

`notes/views.py`
```python
return Note.objects.filter(owner=self.request.user)
```

```python
return Note.objects.filter(owner=self.request.user).order_by('-pub_date')
```

```bash
$ python -m pip install django-wysiwyg
```

```python
INSTALLED_APPS = [
    # ...
    'django_wysiwyg',
    # ...
]

#...
DJANGO_WYSIWYG_FLAVOR = 'ckeditor'
```

```bash
$ cd static
$ wget https://download.cksource.com/CKEditor/CKEditor/CKEditor%204.7.2/ckeditor_4.7.2_full.zip
$ unzip ckeditor_4.7.2_full.zip
$ cd ..
```

`notes/views.py`
```python
from django.core.exceptions import PermissionDenied

...

# In NoteDetail class, override the get() method to raise an
# error if the user tries to view another user's note.

def get(self, request, *args, **kwargs):
    self.object = self.get_object()

    if self.object.owner != self.request.user:
        raise PermissionDenied

    context = self.get_context_data(object=self.object)
    return self.render_to_response(context)
```

`notes/views.py`
```python
from django.views.generic import ListView, DetailView, CreateView, UpdateView

...

class NoteUpdate(LoginRequiredMixin, UpdateView):
    model = Note
    form_class = NoteForm
    template_name = 'notes/form.html'
    success_url = reverse_lazy('notes:index')

    def form_valid(self, form):
       form.instance.pub_date = timezone.now()
       return super(NoteUpdate, self).form_valid(form)
```

`notes/urls.py`
```python
url(r'^(?P<pk>[0-9]+)/edit/$', NoteUpdate.as_view(), name='update'),
```

`templates/notes/form.html`
```html
{% if object %}
<form action="{% url 'notes:update' object.pk %}" method="post" accept-charset="utf-8">
{% else %}
<form action="{% url 'notes:create' %}" method="post" accept-charset="utf-8">
{% endif %}
```

`templates/notes/index.html`
```html
<a href="{% url 'notes:update' note.id %}">Edit</a>
```

`templates/notes/index.html`
```html
<p>
   <a href="{% url 'notes:update' note.id %}">{{ note.title }}</a><br />
   {{ note.body | safe }}
</p>
<hr />
```

`notes/mixins.py`
```python
from .models import Note


class NoteMixin(object):
   def get_context_data(self, **kwargs):
       context = super(NoteMixin, self).get_context_data(**kwargs)

       context.update({
          'notes': Note.objects.filter(owner=self.request.user).order_by('-pub_date'),
       })
       
       return context
```

`notes/views.py`
```python
class NoteCreate(LoginRequiredMixin, NoteMixin, CreateView):
```

```python
class NoteUpdate(LoginRequiredMixin, NoteMixin, UpdateView):

    ...
    def get(self, request, *args, **kwargs):
        self.object = self.get_object()
        
        if self.object.owner != self.request.user:
            raise PermissionDenied

        return super(NoteUpdate, self).get(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        self.object = self.get_object()
        
        if self.object.owner != self.request.user:
            raise PermissionDenied

        return super(NoteUpdate, self).post(request, *args, **kwargs)
```

```bash
$ wget https://github.com/sixfeetup/ElevenNote/raw/master/templates-ch16.zip
$ unzip templates-ch16.zip
```

```bash
cd static
wget https://github.com/sixfeetup/ElevenNote/raw/master/static-elevennote-ch16.zip
unzip static-elevennote-ch16.zip
cd ..
```

`notes/mixins.py`
```python
def check_user_or_403(self, user):
    """ Issue a 403 if the current user is no the same as the `user` param. """
    if self.request.user != user:
        raise PermissionDenied
```

`notes/views.py`
```python
class NoteDelete(LoginRequiredMixin, NoteMixin, DeleteView):
    model = Note
    success_url = reverse_lazy('notes:index')
 
    def post(self, request, *args, **kwargs):
        self.object = self.get_object()
        self.check_user_or_403(self.object.owner)
        return super(NoteDelete, self).post(request, *args, **kwargs)
```

`notes/urls.py`
```python
url(r'^(?P<pk>[0-9]+)/delete/$', NoteDelete.as_view(), name='delete'),
```

`templates/notes/form.html`
```html
{% if object %}
  <form action="{% url 'notes:delete' object.pk %}" method="post" id="delete-note-form">
    {% csrf_token %}
    <a class="btn btn-default" id="delete-note">
      <span class="glyphicon glyphicon-trash" aria-hidden="true"></span>
    </a>
  </form>
{% endif %}
```

![](/assets/Screen Shot 2017-09-16 at 16.08.00.png)


`notes/views.py`
```python
def get_success_url(self):
    return reverse('notes:update', kwargs={
        'pk': self.object.pk
    })
```

### Continuous Integration с CircleCI


### Добавляем API с помощью DRF

```bash
$ pip install djangorestframework
$ python manage.py api
```

`config/settings/base.py`
```python
INSTALLED_APPS = [
    # ...
    'rest_framework',
    # ...
    'api',
]
```

`api/base/serializers.py`
```python
from rest_framework import serializers

from notes.models import Note

class NoteSerializer(serializers.ModelSerializer):

    class Meta:
        model = Note
        fields = ('id', 'title', 'body', 'pub_date')
```
