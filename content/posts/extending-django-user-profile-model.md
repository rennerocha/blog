---
title: "Extending built-in Django User with a Profile Model"
date: 2023-04-24T19:28:40-03:00
tags: ["django", "user", "python"]
draft: false
---

The [user authentication](https://docs.djangoproject.com/en/4.2/topics/auth/#user-authentication-in-django) system provided by Django is extremely powerful
and handles most of the authentication (Am I who I say I am?) and authorization (Am I
authorized to do what I want to do?) needs of an web project. User accounts, groups
and permissions, methods for handling passwords securely are part of this system.

Generally, this is adequate for most projects; however, there are situations where it becomes necessary to modify the behavior of a **User** or alter how their data is stored in the database. It should be noted that modifying the authorization and/or authentication process of a **User** will not be covered in this post.

For the people that prefers video content you can also watch me explaining
the content of this post in the following video (**but in Portuguese**):

{{< peertube "https://peertube.lhc.net.br/videos/embed/20a424c4-2dcc-422a-be77-5f8a1fced27c" >}}

# Extending with an extra Model

The simplest way to extend your **User** is to create a new model with a
[OneToOneField](https://docs.djangoproject.com/en/4.2/ref/models/fields/#django.db.models.OneToOneField) relation to the default **User** model. This
model will contain all extra fields that extends your **User**.

If we are satisfied by the default of Django **User** model and just want
to add extra fields (like a user profile), this is the easiest and simpler
solution.

As an example, to create a user profile storing the date of birth and phone
number of a **User** we could use the model:

{{<highlight python>}}
# myproject/userprofile/models.py
from django.db import models

class Profile(models.Model):
    user = models.OneToOneField(
        "auth.User", on_delete=models.CASCADE, related_name="profile"
    )
    date_of_birth = models.DateField(null=True)
    phone_number = models.CharField(max_length=32, blank=True)

    def __str__(self):
        return f"Profile of '{self.user.username}'"
{{</highlight>}}

However there are some caveats with this approach that we need to keep in mind
to avoid unexpected issues such as:

- A **new table** is created, so retrieving data from both tables will
  require more queries;
  - The use of the [select_related](https://docs.djangoproject.com/en/4.2/ref/models/querysets/#django.db.models.query.QuerySet.select_related) function can
    resolve performance issues caused by the new table. However, if we overlook this aspect, we may encounter unforeseen problems;
- A **Profile** instance is not created automatically when you create a new **User**.
  We can solve this with a [post_save](https://docs.djangoproject.com/en/4.2/ref/signals/#post-save) signal handler;
  - When multiple users are created simultaneously using the [bulk_create](https://docs.djangoproject.com/en/4.2/ref/models/querysets/#django.db.models.query.QuerySet.bulk_create) method, [post_save](https://docs.djangoproject.com/en/4.2/ref/signals/#post-save) is not triggered, which may result in users being created without a **Profile**.
- We need to do some extra work if we want to have these fields added to the
  **User** details in Django Admin

To create a new **Profile** when a new **User** is created we need
to catch [post_save](https://docs.djangoproject.com/en/4.2/ref/signals/#post-save)
signal:

{{<highlight python>}}
# myproject/userprofile/signals.py
from django.db.models.signals import post_save
from django.dispatch import receiver
from django.contrib.auth.models import User

from userprofile.models import Profile

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, *args, **kwargs):
    # Ensure that we are creating an user instance, not updating it
    if created:
        Profile.objects.create(user=instance)
{{</highlight>}}

{{<highlight python>}}
# myproject/userprofile/apps.py
from django.apps import AppConfig

class UserprofileConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "userprofile"

    def ready(self):
        # Register your signal here, so it will be imported once
        # when the app is ready
        from userprofile import signals  # noqa
{{</highlight>}}

Running the application, we will ensure that every time we create
a new **User**, a related **Profile** will be created as well:

{{<highlight python>}}
In [1]: from django.contrib.auth.models import User
   ...: from userprofile.models import Profile

In [2]: Profile.objects.all()
Out[2]: <QuerySet []>

In [3]: user = User.objects.create(username="myuser", email="email@myuser.com")

In [4]: Profile.objects.all()
Out[4]: <QuerySet [<Profile: Profile of 'myuser'>]>
{{</highlight>}}

As mentioned before, when creating instances in bulk, the signal will
not be emited, so the **Profile** will not be automatically created:

{{<highlight python>}}
In [5]: users = User.objects.bulk_create(
   ...:     [
   ...:         User(username="First user", email="user1@user1.com"),
   ...:         User(username="Second user", email="user2@user2.com"),
   ...:     ]
   ...: )

In [6]: users
Out[6]: [<User: First user>, <User: Second user>]

In [7]: Profile.objects.all()
Out[7]: <QuerySet [<Profile: Profile of 'myuser'>]>
{{</highlight>}}

One limitation of this approach is that we are not allowed to have
required fields in the user profile without providing a default
value.

As an example, if we require that `date_of_birth` is mandatory and
our **Profile** model is like the following:

{{<highlight python>}}
# myproject/userprofile/models.py
from django.db import models

class Profile(models.Model):
    user = models.OneToOneField(
        "auth.User", on_delete=models.CASCADE, related_name="profile"
    )
    date_of_birth = models.DateField()  # date_of_birth IS REQUIRED
    phone_number = models.CharField(max_length=32, blank=True)

    def __str__(self):
        return f"Profile of '{self.user.username}'"
{{</highlight>}}

When creating a new user, our [post_save](https://docs.djangoproject.com/en/4.2/ref/signals/#post-save) handler will fail:

{{<highlight python>}}
In [1]: from django.contrib.auth.models import User
   ...: from userprofile.models import Profile

In [2]: user = User.objects.create(username="myuser", email="email@myuser.com")
---------------------------------------------------------------------------
IntegrityError                            Traceback (most recent call last)
(...)
IntegrityError: NOT NULL constraint failed: userprofile_profile.date_of_birth
{{</highlight>}}

Keeping in mind these caveats, extending the **User** model through
this method is a simple and efficient solution that can meet the
needs of many web applications.
