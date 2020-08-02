---
title: Understanding Django REST Framework and Model.full_clean()
date: 2018-11-24 17:46
layout: post
---

If you are new to [Django REST Framework](https://www.django-rest-framework.org/) it may come as a surprise to you that `Model.full_clean()` is not run as part of `ModelSerializer`'s validation routine. This is a departure from something like `ModelForm` which does this for you. It's an easy mistake to make given how similar the two feel.

We can verify this by looking at the source code for [`ModelSerializer.is_valid()`](http://www.cdrf.co/3.7/rest_framework.serializers/ModelSerializer.html#is_valid).

This is due to a (in my opinion, extremely opinionated and not very discoverable) Django REST Framework design decision, introduced in Django REST Framework 3.0.

[Django REST Framework 3.0 announcement blog post](https://www.django-rest-framework.org/community/3.0-announcement/#differences-between-modelserializer-validation-and-modelform):

> We no longer use the `.full_clean()` method on model instances, but instead perform all validation explicitly on the serializer. This gives a cleaner separation, and ensures that there's no automatic validation behavior on `ModelSerializer` classes that can't also be easily replicated on regular `Serializer` classes.
>
> The  `.clean()` method will not be called as part of serializer validation, as it would be if using a `ModelForm`.

A workaround is provided, simply override your serialiser's `validate()` method to call `Model.clean()` yourself:

{% highlight python %}
def validate(self, attrs):
    instance = ExampleModel(**attrs)
    instance.clean()
    return attrs
{% endhighlight %}

But it's presented with a pretty big caveat that this isn't the Right Way™ to do things:

> Again, you really should look at properly separating the validation logic out of the model method if possible, but the above might be useful in some backwards compatibility cases, or for an easy migration path.

### But why?

Django REST Framework contributor Xavier Ordoquy further justifies this design decision [in reply to a Stack Overflow question](https://stackoverflow.com/a/32836543/729785). The real takeaway was Xavier referencing a pretty interesting and thoughtful [blog post](https://www.dabapps.com/blog/django-models-and-encapsulation/) by Django REST Framework author Tom Christie, where he describes a design pattern that allows for this level of separation.

Tom's **own** summary of his blog post is below:

> Never write to a model field or call `save()` directly. Always use model methods and manager methods for state changing operations.
>
> The convention is more clear-cut and easier to follow that "Fat models, thin views", and does not exclude your team from laying an additional business logic layer on top of your models if suitable.
>
> Adopting this as part of your formal Django coding conventions will help your team ensure a good codebase style, and give you confidence in your application-level data integrity.

I encourage you to read the whole post as it contains a few examples to get your head around the pattern.

Tom's suggestions are entirely reasonable implementations of some core software engineering principles ([separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns), [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).etc). However it's a bit of a departure from a few of Django's in-built features, as well as processes outlined in the Django documentation. The main / only offender is `ModelForm` and by extension other technologies that use `ModelForm` under the hood (generic class-based views, the Django admin interface.etc). This is because `ModelForm` handles model state changes for you, **which bypasses your service layer**.

This is somewhat justified in [the Django documentation for `django.contrib.admin`](https://docs.djangoproject.com/en/2.1/ref/contrib/admin/index#module-django.contrib.admin) (which heavily utilises `ModelForm` behind the scenes):

> The admin has many hooks for customization, but beware of trying to use those hooks exclusively. If you need to provide a more process-centric interface that abstracts away the implementation details of database tables and fields, then it’s probably time to write your own views.

There are situations in which overriding `ModelForm` to use your state change routines may still make sense. Even if you override `ModelForm`'s default save behaviour it still has a lot to offer (mainly dynamic field construction).

### Implementing the correct solution

The Django REST Framework 3.0 announcement blog post recommends overriding `ModelSerializer.validate()` - which is an explicit validation step separate from the actual state change. This slightly goes against the examples provided in Tom's blog post, where no provision was made to have the service layer perform data validation separate from the state change itself.

I am of the opinion that it is not within the spirit of Tom's blog post to have **public** validation routines defined within your services layer, but defining validation logic within `ModelSerializer.validate()` is undoubtedly *worse* for DRY / separation of concerns. A more elegant (but still not perfect) solution is to completely override [`ModelSerializer.create()`](http://www.cdrf.co/3.7/rest_framework.serializers/ModelSerializer.html#save) / [`ModelSerializer.update()`](http://www.cdrf.co/3.7/rest_framework.serializers/ModelSerializer.html#update) and hook into your services layer there. From what I can tell, there's no harm in performing validation outside of `validate()` - it is merely provided an easily overridable hook to apply validation without overriding the state change operation.

The only spanner in the works (which applies no matter where you perform your validation) is that Django REST Framework [expects](https://www.django-rest-framework.org/community/3.0-announcement/#using-serializersvalidationerror) you to raise its own `serializers.ValidationError` instead of `django.core.exceptions.ValidationError`. It's up to you whether you do this in your services layer, or if you `try` / `catch` inside of `create()` / `update()` and raise Django REST Framework's `ValidationError` from there. I'd opt for the latter.
