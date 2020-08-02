---
title: Django permissions in the real world
date: 2020-05-29 16:11:00 +08:00
layout: post
---

Django's [built-in permissions system](https://docs.djangoproject.com/en/1.11/topics/auth/default#permissions-and-authorization) is deceptively simple. It is a very good turnkey solution for basic content-driven websites, or web applications without much of a permissions mechanic. However it is not (and is not meant to be) a complete solution for more complex requirements.

Django's inbuilt permissions system does not support object-level permissions, this is to say that Django only allows you to *globally* grant a user/group a permission. Consider [the official Django tutorial](https://docs.djangoproject.com/en/3.0/intro/tutorial01/)'s 'polls' application. You cannot—say—grant a particular access to only *certain* `Question`s (e.g. only `Question`s the user themselves created), as opposed to the all `Question`s that are—or ever will be—in the database.

### QuerySet filtering

In the aformentioned unrealistically sterile and simple example, you could do something like the following:

 {% highlight python %}
from django.views import generic

from . import models


class UpdateView(generic.UpdateView):

    model = models.Question

    def get_queryset(self, *args, **kwargs):
        queryset = super().get_queryset(*args, **kwargs)
        return queryset.filter(created_by=self.request.user)
{% endhighlight %}

In my opinion, this is fine. I have done this, and will continue to do this in some cases. Software development is all about weighing up costs and benefits, so before doing something like the above, consider the following:

1. How scalable and DRY is this?
2. How do you check the permissions for a single instance, instead of a `QuerySet`?
3. Do you want to indicate to a user that they do not have permission to access this object (instead of pretending that it doesn't exist)? If so, how will you do it?
4. Does this cover all the edge cases?
5. How will you check for permissions in other contexts, like templates?


These are all solveable problems, however they are largely solved problems, and if your application has a lot of permissions system involvement, it may be worth looking into something that addresses these concerns.


### Django's object-level permissions API

Django does not fully address object-level permissions, but it acknowledges the usecase by providing an object-level permissions API that can be implemented by third parties.

Django's authentication and authorisation system is pluggable. For instance, when you call `user.has_perm()`, Django (more specifically [`PermissionsMixin`](https://docs.djangoproject.com/en/3.0/topics/auth/customizing/#django.contrib.auth.models.PermissionsMixin.has_perm)) asks each of the **authentication backends** you have configured in [`settings.AUTHENTICATION_BACKENDS`](https://docs.djangoproject.com/en/3.0/ref/settings#authentication-backends) if the permission should be granted. Each authentication backend is given an opportunity to say "yes", "no", or "I don't have an opinion". By default, the only enabled backend is Django's in-built authentication backend, [`ModelBackend`](https://docs.djangoproject.com/en/3.0/ref/contrib/auth/#django.contrib.auth.backends.ModelBackend).

Note that 'authentication backend' is a slight misnomer as authentication backends can handle both authentication (i.e. "is this user who they say they are?") **and/or** authorisation (i.e. "what resources should this person be allowed to access?").


### django-guardian

[`django-guardian`](https://github.com/django-guardian/django-guardian) is by my estimate the most popular Django object-level permissions application. It provides a [custom authentication backend](https://django-guardian.readthedocs.io/en/stable/api/guardian.backends.html#objectpermissionbackend) which means that it makes itself available through standard Django mechanics like [`has_perm()`](https://docs.djangoproject.com/en/3.0/topics/auth/customizing/#django.contrib.auth.models.PermissionsMixin.has_perm).

For instance, if you installed `django-guardian`'s custom authentication backend and did the following, `django-guardian` would be queried.

{% highlight python %}
from django.core.exceptions import PermissionDenied
from django.shortcuts import get_object_or_404

def myview(request, pk):
    question = get_object_or_404(Question, pk=pk)
    if not request.user.has_perm("polls.read_question", question):
        raise PermissionDenied()
{% endhighlight %}

In `django-guardian`, object permissions are recorded in the database. Just as you can assign `Permission`s to `User`s or `Group`s using Django's standard permissions system, you can assign `Permission`s **for a specfiic object** to `User`s or `Group`s.

``django-guardian`` is well-documented so I will not rehash its API here. Suffice to say that the API looks something like this:

{% highlight python %}
from django.contrib.auth.models import User
from guardian.shortcuts import assign_perm
from polls.models import Question

user = User.objects.create(username='John Smith')
question = Question.objects.create()
user.has_perm('answer_question', question)  # Returns False
assign_perm('answer_question', user, question)
joe.has_perm('answer_question', question)  # Returns True
{% endhighlight %}

Under the hood, `django-guardian` is fairly simple. [Unless instructed otherwise](https://django-guardian.readthedocs.io/en/stable/userguide/performance.html#direct-foreign-keys), it operates using two models: [`UserObjectPermission`](https://django-guardian.readthedocs.io/en/stable/api/guardian.models.html#userobjectpermission) and [`GroupObjectPermission`](https://django-guardian.readthedocs.io/en/stable/api/guardian.models.html#groupobjectpermission), linking users/groups + permissions to model instance using [generic foreign keys](https://docs.djangoproject.com/en/3.0/ref/contrib/contenttypes#s-id1).


### The pros and cons of `django-guardian`

`django-guardian` is a solid implementation of what the project sets out to achieve, which is to allow object-level permissions to be configured in a database. However, what `django-guardian` sets out to achieve has an inherent set of pros and cons.

As `django-guardian`, permissions are configured within the context of the database. This means that permissions can be determined within database queries. This is beneficial for doing things like filtering a `QuerySet` such that it only contains records for which a given user has a given permission.

Continuing from our example. Say that we wanted to create a view that displayed a list of `Question`s that a user is allowed to read:

{% highlight python %}
from guardian.shortcuts import get_objects_for_user

def question_list(request):
    questions = get_objects_for_user(request.user, "questions.view_question")
    # ... rendering logic ...
{% endhighlight %}

`django-guardian`'s inherent shortfall (at last in the contexts that I've tried to use it) is that my application will typically have its own existing 'source of truth' to determine which users should receive which permissions on which objects. I may be showing my shortcomings as a developer by saying that the cognitive load of keeping `django-guardian` object permission records in sync with the rest of my application is mentally taxing. This is the typical [cache invalidation](https://en.wikipedia.org/wiki/Cache_invalidation) problem, as `django-guardian`'s permissions table is effectively acting as a cache for your other business logic. As this is one of the two infamously hard problems in computing, I feel that I am not alone.

Within the context of our polls application, imagine that business rules dictate that users should only be allowed to change `Question`s that they wrote.

You could keep the `django-guardian` object permissions records in sync with the 'truth' (that is, which questiosn are owned by which users?). Honestly on its surface this doesn't sound too hard...but that's the temptingly misleading simplicity of cache invalidation. You could hook `Model.save()`, write a `post_save` signal, or any number of things. Unless you are extremely disciplined, you may find that changes fall through the cracks. For instance, did you know that bulk updates do not trigger `Model.save()` or the `pre_save`/`post_save` signals? Do you trust yourself to remember this at 4PM on the last day of your sprint?

You may instead opt for a hybrid of `django-guardian` and plain `QuerySet` filtering. Using `django-guardian` to determine 'manually configured' object permissions, and `QuerySet` filtering for things that can be determined elsewhere in the database state. I do not recommend this for most circumstances. This breaks the contract that `has_perm()` offers. You may use `has_perm()` but forget to perform your manual `QuerySet` filtering, or vice versa. You could write an abstraction, but there's a lot to be gained from being able to use Django's permissions APIs.
