# Создание и настройка проекта Django

### Установка зависимостей
```
python -m venv venv  
source venv/bin/activate  # Linux / Mac  
venv\Scripts\activate     # Windows  
  
pip install django psycopg2-binary
```
### Создание проекта
```
django-admin startproject config  
cd config  
 
python manage.py startapp blog
```
Структура будет примерно такая:

config/  
 ├── config/  
 ├── blog/  
 ├── manage.py

### Подключаем приложение

В `config/settings.py`:
```
INSTALLED_APPS = [  
    ...  
    'blog',  
]
```
---

# 🐘 2. Подключение PostgreSQL

### Установка PostgreSQL

- Linux: `sudo apt install postgresql`
- Mac: `brew install postgresql`
- Windows: скачать с официального сайта

### Создание БД
```
CREATE DATABASE myproject;  
CREATE USER myuser WITH PASSWORD 'mypassword';  
ALTER ROLE myuser SET client_encoding TO 'utf8';  
ALTER ROLE myuser SET default_transaction_isolation TO 'read committed';  
ALTER ROLE myuser SET timezone TO 'UTC';  
GRANT ALL PRIVILEGES ON DATABASE myproject TO myuser;
\c myproject
GRANT ALL ON SCHEMA public TO myuser;
ALTER SCHEMA public OWNER TO myuser;
```
### Настройка Django

В `settings.py`:
```
DATABASES = {  
    'default': {  
        'ENGINE': 'django.db.backends.postgresql',  
        'NAME': 'myproject',  
        'USER': 'myuser',  
        'PASSWORD': 'mypassword',  
        'HOST': 'localhost',  
        'PORT': '5432',  
    }  
}
```
### Применяем миграции
```
python manage.py migrate
```
---

# 🧱 3. Базовые модели

Открываем `blog/models.py`:
```
from django.db import models  
  
  
class Category(models.Model):  
    name = models.CharField(max_length=255)  
    slug = models.SlugField(unique=True)  
  
    def __str__(self):  
        return self.name  
  
  
class Post(models.Model):  
    title = models.CharField(max_length=255)  
    slug = models.SlugField(unique=True)  
    content = models.TextField()  
    created_at = models.DateTimeField(auto_now_add=True)  
    published = models.BooleanField(default=False)  
  
    def __str__(self):  
        return self.title  
  
  
class Article(models.Model):  
    title = models.CharField(max_length=255)  
    category = models.ForeignKey(  
        Category,  
        on_delete=models.CASCADE,  
        related_name='articles'  
    )  
    content = models.TextField()  
    created_at = models.DateTimeField(auto_now_add=True)  
    published = models.BooleanField(default=False)  
  
    def __str__(self):  
        return self.title
```
---

### Создаем миграции
```
python manage.py makemigrations  
python manage.py migrate
```
---

# 🔐 4. Админка Django

### Создание суперпользователя
```
python manage.py createsuperuser
```
---

### Регистрируем модели

В `blog/admin.py`:
```
from django.contrib import admin  
from .models import Post, Category, Article  
  
  
@admin.register(Post)  
class PostAdmin(admin.ModelAdmin):  
    list_display = ('title', 'created_at', 'published')  
    list_filter = ('published',)  
    prepopulated_fields = {'slug': ('title',)}  
  
  
@admin.register(Category)  
class CategoryAdmin(admin.ModelAdmin):  
    prepopulated_fields = {'slug': ('name',)}  
  
  
@admin.register(Article)  
class ArticleAdmin(admin.ModelAdmin):  
    list_display = ('title', 'category', 'created_at', 'published')  
    list_filter = ('category', 'published')
```
---

### Запуск сервера
```
python manage.py runserver
```
Открыть:

http://127.0.0.1:8000/admin/
