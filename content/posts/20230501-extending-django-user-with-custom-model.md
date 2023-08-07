---
title: "Extending built-in Django User with a custom Model"
date: 2023-05-01T19:28:40-03:00
tags: ["django", "user", "python"]
slug: extending-django-user-with-custom-model
---

The [user authentication](https://docs.djangoproject.com/en/4.2/topics/auth/#user-authentication-in-django) system provided by Django is extremely powerful
and handles most of the authentication (Am I who I say I am?) and authorization (Am I
authorized to do what I want to do?) needs of an web project. User accounts, groups
and permissions, methods for handling passwords securely are part of this system.

Generally, this is adequate for most projects; however, there are situations where it becomes necessary to modify the behavior of a **User** or alter how their data is stored in the database. It should be noted that modifying the authorization and/or authentication process of a **User** will not be covered in this post.

Creating a user profile model is one solution to extend and modify the behavior of a **User**, as [discussed before]({{< ref "20230424-extending-django-user-profile-model" >}}), but that approach has some caveats that are solved with the use of a
custom user model.

# Custom User model

We can tell Django which is the model that will replace the built-in
`django.contrib.auth.models.User` model in the system. This is done by
providing a value to `AUTH_USER_MODEL` in your project.

{{<highlight python>}}
# myproject/myproject/settings.py
AUTH_USER_MODEL = "users.CustomUser"
{{</highlight>}}

> Usually I create a `users` application to group all custom user related
> code to keep everything in the same context

It is highly recommended that you do this configuration at the beginning
of the project, as after you have created your database tables, it will
require you to fix your schema, moving data between tables and reapplying
some migrations manually (see [#25313](https://code.djangoproject.com/ticket/25313)
for an overview of the steps involved).

Now we need to create our custom model.

This new user model can handle different authentication and authorization
schemes, can use different fields (e.g. email) to identify the user or
any other requirement that is not satisfied using the default Django user
model.

## Adding new fields to default User model

If the default **[User](https://docs.djangoproject.com/en/4.2/ref/contrib/auth/#django.contrib.auth.models.User)** model has everything we need and we want
to add some extra profile fields keeping authentication and authorization as
it is, create a custom model inheriting from `AbstractUser` would be enough.

{{<highlight python>}}
# myproject/users/models.py
from django.contrib.auth.models import AbstractUser

class CustomUser(AbstractUser):
    date_of_birth = models.DateField(null=True)
{{</highlight>}}

{{<highlight python>}}
# myproject/myproject/settings.py
AUTH_USER_MODEL = "users.CustomUser"
{{</highlight>}}

As we will not create a new table to store the user related information, there is
no need for additional database queries to retrieve related models. We also don't
need to worry if a related model is created or not reducing the complexity of
the system.

## AbstractBaseUser

If we want to use a different field as the identifier (other than `email`),
use a different authorization system and/or having more control on what
fields we have in our user model, a custom model inheriting from
`AbstractBaseUser` is the choice.

`AbstractBaseUser` is the core implementation of a **User**. It has a
`password` field and methods to store and validate it securely. A
`last_login` datetime field and everything else is up to us to provide.

If using the default authentication backend, our model must have a single unique
field that will be the identifier of our user such as email address, username or
any other unique attribute.

You configure that attribute using `USERNAME_FIELD` that defines what is
the field that will uniquely identify the user.





`EMAIL_FIELD`

`REQUIRED_FIELDS`

`is_active` is an attribute on `AbstractBaseUser` defaulting to `True`

As an example, we will define a custom user that will use the email address
as the identifier, has the date of birth as one required field can store
the phone number and has a flag field in the database to indicate if the user
is active or not.

{{<highlight python>}}
# myproject/users/models.py
from django.contrib.auth.models import AbstractBaseUser

class CustomUser(AbstractBaseUser):
    USERNAME_FIELD = "email"
    EMAIL_FIELD = "email"
    REQUIRED_FIELDS = ["date_of_birth", ]

    date_of_birth = models.EmailField(unique=True)
    email = models.EmailField(unique=True)
    is_active = models.BooleanField(default=True)
    phone_number = models.CharField(max_length=16, blank=True)
{{</highlight>}}



FROM DOCS Authentication backends provide an extensible system for when a username and password stored with the user model need to be authenticated against a different service than Django’s default.

TODO: list all fields that are created with this

Any other fields needs to be created.

 is_active is a property set to True, but you can define if you want

USERNAME_FIELD is needed identifier unique = True
EMAIL_FIELD to specifi what is the email field
REQUIRED_FIELDS fields required when createsuperuser

If we want to use `email` as the identifier of an user,
we need to set `USERNAME_FIELD` property

{{<highlight python>}}
# myproject/users/models.py
class CustomUser(AbstractBaseUser):
    USERNAME_FIELD = "email"

    email = models.EmailField()
{{</highlight>}}

One drawback is you need to specify a Custom Manager implementing
two methods: `create_user` and `create_superuser`. This is required
so we willll be able to run `python manage.py createsuperuser`
and `python manage.py create-user`.

If some fields (TODO list what fields) from `ABstractUser` are there, you can
use the built-in UserManager
FROM DOCS username, email, is_staff, is_active, is_superuser, last_login, and date_joined fields the same as Django’s default user, you can install Django’s UserManager

{{<highlight python>}}
# myproject/users/models.py
class CustomUser(AbstractBaseUser):
    USERNAME_FILED = "email"

    email = models.EmailField()

    objects = UserManager()
{{</highlight>}}

If you don't want to use Django Admin (very unlikely), you're done,
and you can create/manage your users and authentication backend will
be able.

However, if you want to use this custom user model with Django admin, there are
some extra steps.

We need  some fields and methods:
`is_staff` so you have access to django admin
`is_active` inactive users can't access django admin

to manage permissions to edit registered models
`has_perm`
`has_module_perms()`
***we don't need to use the built-in permission system (see PermissionMixin)***

And `UserCreationForm`< `UserChangeForm` and `PasswordResetForm` (if we don't have email field)
needs to be customized so we can use that.

Also we need a custom UserAdmin to use our custom forms.

This is a lot of things, and sometimes difficult to remember. The docs
are ok to describe the process, but IMO this should be much simpler :-P

If by any chance you want to use django model permission, you can
use PermissionMixin, so you have access to groups, etc (to be honest, I
never saw anything using permission, but groups)

















Sometimes the username may not be the way you want to identify an user
(an email or social number would be more suitable for your project)
or we have different requirements for authorization and permission. In
those cases creating a custom user model is a option.

This has advatanges over the user of a user profile model and solve some
of the caveats of that approach such as:

- We won't need an extra database table to store your custom fields, so
  there is no performance problems to access them;
- As the table that we store the User data required for Django authentication
  is the same, we don't need to bother if your profile fields will be available
  or not

When creating a custom user model, we have two different options
If you are going to this path, you can create your custom user
inheriting  from two different classes:






If using Django Admin, we also need to register this new model.

{{<highlight python>}}
# myproject/users/admin.py
from django.contrib import admin
from django.contrib.auth.admin import UserAdmin
from .models import CustomUser

admin.site.register(CustomUser, UserAdmin)
{{</highlight>}}