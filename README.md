Step-1 Setting up

To work with GraphQL with Django we need to install Graphene an open-source package that is pretty cool and gives us leverage to utilize amazing features to define our GraphQL API, please do check out their docs for more details — https://docs.graphene-python.org/projects/django/en/latest/

To install Django run the following command using pip:
```bash
pip install Django
```
To install Graphene run the following command using pip:

```bash
pip install graphene-django
```

So we installed all the packages, now let’s create a Django project by the following command:

```bash
django-admin startproject books
```

Now books is our parent directory, where we will configure all our required packages dependencies in books/settings.py file.

Let’s generate a application; resaleapp, it’s basically a simple application to resale books by different sellers:

```bash
python3 manage.py startapp resaleapp
```

In books/settings.py file in books directory include the following the newly generated app and graphene_django dependency.

```bash
INSTALLED_APPS = [
'django.contrib.admin',
'django.contrib.auth',
'django.contrib.contenttypes',
'django.contrib.sessions',
'django.contrib.messages',
'django.contrib.staticfiles',
'graphene_django',
'resaleapp'
]
```

You can use any database to develop this application, Let’s define the configuration in books/settings.py:

```bash
DATABASES = {
  "default": {
      "ENGINE": "django.db.backends.postgresql",
      "NAME": "books",
      "HOST": "localhost,
      "PORT": "5432",
      "USER": "postgres",
      "ATOMIC_MUTATIONS": True,
      "PASSWORD": os.environ['POSTGRES_PASSWORD'] # Your local DB password
      }
}
```

Add GraphQL in books/settings.py

```bash
GRAPHENE = {
  'SCHEMA': 'resaleapp.schema.schema'
}
```

Configure your media path to store the media/static files, In books/settings.py

```bash
MEDIA_URL = ‘/media/’
MEDIA_ROOT = os.path.join(BASE_DIR, ‘media’)
STATICFILES_DIRS = (os.path.join(BASE_DIR, ‘static’), )
```

Now, In resale_app/urls.py, include these libraries for development environment

```bash
from django.urls import path
from resaleapp.schema import schema
from  import views
from django.views.decorators.csrf import csrf_exempt
from graphene_file_upload.django import FileUploadGraphQLViewurlpatterns = [path('graphql/',
               csrf_exempt(FileUploadGraphQLView.as_view(graphiql=True,
               schema=schema)))]
```

Finally, Include the resaleapp url and wrap it media configuration in the parent directory urls.pyfile in books/urls.py

```bash
from django.contrib import admin
from django.urls import path, include
from django.views.decorators.csrf import csrf_exempt
from graphene_file_upload.django import FileUploadGraphQLView
from resaleapp.schema import schema
from graphene_django.views import GraphQLView
from django.conf.urls.static import static
from django.conf import settingsurlpatterns = [path('admin/', admin.site.urls), path('',
               include('resaleapp.urls'))] + static(settings.MEDIA_URL,
        document_root=settings.MEDIA_ROOT)
```

Step -2 Run your Application

Once you have all these setups up in the application, we are good to start the server on http://localhost:8000/graphql/ using the following command.

```bash
python3 manage.py runserver
```

If you get an error, its because we haven’t created schema.py and trying to import in resaleapp/urls.py (remove it for now)

```bash
urlpatterns = [path('graphql/',
               csrf_exempt(FileUploadGraphQLView.as_view(graphiql=True)))]
```

Step -3 Create your models

```bash
from django.db import models
import uuid 
import datetime
import os
# Create your models here.
def filepath(request, filename): # File Path for your uploaded media    
    old_filename = filename
    timeNow = datetime.datetime.now().strftime('%Y%m%d%H:%M:%S')
    filename = '%s_%s' % (timeNow, old_filename)
    return os.path.join('media/', filename)

class User(models.Model):    
    id = models.UUIDField(primary_key=True, default=uuid.uuid4,
                          editable=False, unique=True)
    name = models.CharField(max_length=512)
    location = models.CharField(null=True, max_length=512)
    email = models.EmailField(null=True)
    phone_no = models.CharField(max_length=10, null=True)    
    def __str__(self):
        return self.name
    
class Book(models.Model):    
    id = models.UUIDField(primary_key=True, default=uuid.uuid4,
                          editable=False, unique=True)
    book_name = models.CharField(max_length=512)
    author = models.CharField(max_length=512)
    price = models.CharField(max_length=512)
    book_image = models.FileField(upload_to=filepath, unique=True)

class BookUser(models.Model):    
    id = models.UUIDField(primary_key=True, default=uuid.uuid4,
                          editable=False, unique=True)
    user = models.ForeignKey(User, null=True, on_delete=models.CASCADE)
    book = models.ForeignKey(Book, null=True, on_delete=models.CASCADE)

class Comment(models.Model):    
    id = models.UUIDField(primary_key=True, default=uuid.uuid4,
                          editable=False, unique=True)
    book = models.ForeignKey(Book, null=True, on_delete=models.CASCADE)
    body = models.CharField(max_length=512)
    
class CommentUser(models.Model):    
    id = models.UUIDField(primary_key=True, default=uuid.uuid4,
                          editable=False, unique=True)
    user = models.ForeignKey(User, null=True, on_delete=models.CASCADE)
    comment = models.ForeignKey(Comment, null=True,
                                on_delete=models.CASCADE)
    

    
```

Let’s now migrate the changes to our PostgreSQL database for the models we just created

```bash
python3 manage.py makemigrations
python3 manage.py migrate
```

Hope you don’t get any errors here :)

Step — 4 Lets create our resaleapp/fields.py file.

In general , it is used to convert the data into respective formats for processing.

A Type is a GraphQL object that represents your model, you can customize it to allow you to filter your results based on a set of criteria.

Let’s create a fields file for graphene in the app resaleapp/fields.py

```bash
import graphene
from graphene_django import DjangoObjectType
# from graphene_django.filter import DjangoFilterConnectionField
from .models import User, Book, Comment, BookUser, CommentUser


class BookClass(DjangoObjectType):    
    class Meta:       
        model = Book

class UserClass(DjangoObjectType):    
    book = graphene.List(BookClass)    
    class Meta:      
        model = User    
        def resolve_book(self, info, root):
            book_user = \
                  BookUser.objects.filter(user_id=self.id).values_list('book_id'
                )
            books = Book.object.filter(id__in=book_user)
            return books

class BookUserClass(DjangoObjectType):    
    class Meta:      
        model = BookUser
class CommentClass(DjangoObjectType):    
    class Meta:       
        model = Comment
class CommentUserClass(DjangoObjectType):    
    class Meta:       
        model = CommentUser
```

If you’ve observed, each model that are created have their own field class types defined, it is also allowed to define custom resolvers to each of these fields, we can access it by calling it’s own class connection.

Step — 4 Let’s create our Schemas and Mutations for handling query requests from the client in GraphQL API.

Let’s create a mutation file for graphene in the app resaleapp/mutations.py

Now, let’s define few mutatefunctions to store our data in PostgreSQL.

```bash
import graphene
from graphene_file_upload.scalars import Upload
from .fields import UserClass, BookClass, CommentClass
from .models import User, Book, Comment, BookUser, CommentUser



class createUser(graphene.Mutation):    
    error = graphene.String()
    success = graphene.Boolean()
    user = graphene.Field(UserClass)    
    class Meta:    
        description = 'Add user details'    
    class Arguments:        
        name = graphene.String(required=True)
        email = graphene.String(required=True)
        location = graphene.String()
        phone_no = graphene.String()    
    def mutate(
        self,
        info,
        phone_no=None,
        location=None,
        **kwargs
        ):
        try:
            user = User.objects.create(name=kwargs.get('name'),
                    email=kwargs.get('email'), phone_no=phone_no,
                    location=location)
            return createUser(user=user, success=True)
        except Exception as e:
            return createUser(user=None, success=False, error=e)
        
class createBook(graphene.Mutation):    
    error = graphene.String()
    success = graphene.Boolean()
    book = graphene.Field(BookClass)    
    class Arguments:        
        user_id = graphene.ID(required=True)
        book_name = graphene.String(required=True)
        author = graphene.String(required=True)
        book_image = Upload()
        price = graphene.Int()    
    def mutate(self, info, **kwargs):
        try:
            book = Book.objects.create(book_name=kwargs.get('book_name'
                    ), author=kwargs.get('author'),
                    book_image=kwargs.get('book_image'),
                    price=kwargs.get('price'))
            book_user = BookUser.objects.create(book_id=book.id,
                    user_id=kwargs.get('user_id'))
            return createBook(book=book, success=True)
        except Exception as e:
            return createBook(book=book, success=False, error=e)
        
class createComment(graphene.Mutation):    
    error = graphene.String()
    success = graphene.Boolean()
    comment = graphene.Field(CommentClass)    
    class Arguments:        
        user_id = graphene.ID(required=True)
        book_id = graphene.ID(required=True)
        body = graphene.String(required=True)    
    def mutate(self, info, **kwargs):
        try:
            comment = \
                Comment.objects.create(book_id=kwargs.get('book_id'),
                    body=kwargs.get('body'))
            comment_user = \
                CommentUser.objects.create(user_id=kwargs.get('user_id'
                    ), comment_id=comment.id)
            return createComment(success=True, comment=comment)
        except Exception as e:
            return createComment(success=False, error=e)
        
class delete(graphene.Mutation):    
    error = graphene.String()
    success = graphene.Boolean()    
    class Arguments:        
        user_id = graphene.ID()
        book_id = graphene.ID()
        comment_id = graphene.ID()    
    def mutate(self, info, **kwargs):
        try:
            if kwargs.get('user_id'):
                User.objects.filter(id=kwargs.get('user_id')).delete()
            if kwargs.get('book_id'):
                Book.objects.filter(id=kwargs.get('book_id')).delete()
            if kwargs.get('comment_id'):
                Comment.objects.filter(id=kwargs.get('comment_id'
                        )).delete()
            return delete(success=True)
        except Exception as e:
            return delete(success=False, error=e)

```

For example, We created a UserClass class which is an Object that represents our model, and then a mutate function createUser that creates a record for User model.

Now, lets create a schema file for graphene in the app resaleapp/schema.py

A Schema is essentially a file that converts your models to GraphQL and informs it what they are and how they should be interacted with.

```bash
import graphene
from .fields import UserClass, CommentClass, BookClass
from .models import User, Book, Comment
from .mutations import createUser, createBook, createComment, delete



class Query(graphene.ObjectType):    
    users = graphene.List(UserClass)
    user = graphene.Field(UserClass, id=graphene.ID())
    books = graphene.List(BookClass)
    book = graphene.Field(BookClass, id=graphene.ID())
    comments = graphene.List(CommentClass)
    comment = graphene.Field(CommentClass, id=graphene.ID())
    def resolve_users(root, info):
        return User.objects.all()
    def resolve_user(root, info, id):
        return User.objects.get(id=id)
    def resolve_books(root, info):
        return Book.objects.all()
    def resolve_book(root, info, id):
        return Book.objects.get(id=id)
    def resolve_comments(root, info):
        return Comment.objects.all()
    def resolve_comment(root, info, id):
        return Comment.objects.get(id=id)
    
class Mutation(graphene.ObjectType):
    create_user = createUser.Field()
    create_book = createBook.Field()
    create_comment = createComment.Field()
    delete = delete.Field()

schema = graphene.Schema(query=Query, mutation=Mutation)
```

For example, We created a class UserClass which is an Object that represents our model, and then a resolver resolve_users that returns the value of a field

Awesome, Good job! You all achieved the necessary for building a basic application using Graphene + Django :)

Step — 7 Let’s test our Application at http://localhost:8000/graphql/

To Create/Delete/Update data we use mutation query in GraphQL

Mutate Query:
```bash
mutation{
  createUser(email:"booksapp@gmail.com", phoneNo:"123456789", name:"Sammy", location:"New York"){
    success
    user{
      name,
      phoneNo,
      email,
      location
    }
  }
}
```
Response:
```bash
{
  "data": {
    "createUser": {
      "success": true,
      "user": {
        "name": "Sammy",
        "phoneNo": "123456789",
        "email": "booksapp@gmail.com",
        "location": "New York"
            }
    }
  }
}
```