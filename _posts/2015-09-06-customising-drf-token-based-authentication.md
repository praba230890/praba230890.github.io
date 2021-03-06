---
layout: post
title: Customising Django Rest Framework (DRF) Token Based Authentication
---

In this project we are going to generate REST API's for a mobile client which are 
protected by token based authentication. Unlike normal token based authentication 
provided by DRF, we are not going to use the django default User model for generating and 
authenticating the tokens. Here, I don't know why but our client wants to generate tokens based on the
device mac address and authenticate them by the same. So, in order to achieve this authentication
we are going to use the `authtoken` app in DRF which is responsible for token based authentication.

Ok, lets start a project,

{% highlight bash %}
$ django-admin startproject projectname
{% endhighlight %}

Download the Django Rest Framework source code from Github

{% highlight bash %}
$ git clone https://github.com/tomchristie/django-rest-framework/
{% endhighlight %}

Now copy the authtoken app from django-rest-framework/rest_framwork/ to your project folder

{% highlight bash %}
$ cp -r django-rest-framework/rest_framwork/authtoken /path/to/my/django/project/
{% endhighlight %}

Lets create an app called `device` which is the main app,

{% highlight bash %}
$ python manage.py startapp device
{% endhighlight %}

then create the basic model for the device,

{% highlight python %}
# device/models.py
from django.db import models
# for token based auth, creating tokens automatically for all users
from django.conf import settings
from django.db.models.signals import post_save
from django.dispatch import receiver
from authtoken.models import Token

class Device(models.Model):
    """
       Device model - device app

       fields: mac_address, device_description

       table to store the device details
    """
    mac_address = models.CharField(max_length=45)
    device_description = models.CharField(max_length=64)

    def __unicode__(self):
        return u"%s" % (self.mac_address)

@receiver(post_save, sender=Device)
def create_auth_token(sender, instance=None, created=False, **kwargs):
    if created:
        Token.objects.create(device=instance)
{% endhighlight %}

Here the `create_auth_token` function will create an authentication token for every device that
will be created, automatically by using signals.

Now, I'm going to customise the Token based authentication to use device details (`Device` model) for 
token generation and validation instead of User model.

{% highlight bash %}
(myproject/authtoken)$ vim models.py
{% endhighlight %}

{% highlight python %}
# authtoken/models.py
import binascii
import os
from django.utils import timezone

from django.conf import settings
from django.db import models
from django.utils.encoding import python_2_unicode_compatible

@python_2_unicode_compatible
class Token(models.Model):
    """
    The authorization token model based on Mac Address (not user)
    """
    key = models.CharField(max_length=40, primary_key=True)
    device = models.OneToOneField('device.Device', related_name='auth_token')
    created = models.DateTimeField(auto_now_add=True)

    def save(self, *args, **kwargs):
        if not self.key:
            self.key = self.generate_key()
        return super(Token, self).save(*args, **kwargs)

    def generate_key(self):
        return binascii.hexlify(os.urandom(20)).decode()

    def __str__(self):
        return self.key
{% endhighlight %}

Now we need to change the DRF authtoken views and serializer to obtain token based on 
device `mac_address` instead of the `username` and `password` combination used by default.

{% highlight python %}
# authtoken/views.py
from rest_framework import parsers, renderers
from .models import Token
from .serializers import AuthTokenSerializer
from rest_framework.response import Response
from rest_framework.views import APIView

class ObtainAuthToken(APIView):
    throttle_classes = ()
    permission_classes = ()
    parser_classes = (parsers.FormParser, parsers.MultiPartParser, parsers.JSONParser,)
    renderer_classes = (renderers.JSONRenderer,)
    serializer_class = AuthTokenSerializer

    def post(self, request):
        serializer = self.serializer_class(data=request.data)
        serializer.is_valid(raise_exception=True)
        device = serializer.validated_data['device']
        token, created = Token.objects.get_or_create(device=device)
        return Response({'token': token.key})
{% endhighlight %}

{% highlight python %}
# authtoken/serializers.py
from django.contrib.auth import authenticate
from django.utils.translation import ugettext_lazy as _

from rest_framework import serializers

from device.models import Device

class AuthTokenSerializer(serializers.Serializer):
    mac_address = serializers.CharField()

    def validate(self, attrs):
        mac_address = attrs.get('mac_address')

        if Device.objects.filter(mac_address=mac_address).exists():
            device = Device.objects.get(mac_address=mac_address)
        else:
            msg = _('Device not registered')
            raise serializers.ValidationError(msg)

        attrs['device'] = device
        return attrs
{% endhighlight %}

and finally the admin.py file,

{% highlight python %}
# authtoken/admin.py
from django.contrib import admin

from .models import Token


class TokenAdmin(admin.ModelAdmin):
    list_display = ('key', 'device', 'created')
    fields = ('device',)
    ordering = ('-created',)

admin.site.register(Token, TokenAdmin)
{% endhighlight %}

Now we are ready to roll, but before that we need to 

 - add these `authtoken` and `device` apps to installed apps in settings file
 - add url for obtaining tokens
 - run makemigrations and migrate

{% highlight python %}
# projectname/settings.py
INSTALLED_APPS = (
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework', # if you haven't installed DRF, do: pip install djangorestframework
    'device',
    'authtoken',
)
{% endhighlight %}

{% highlight python %}
# projectname/urls.py
from django.conf.urls import include, url
from django.contrib import admin
from authtoken.views import ObtainAuthToken

urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^token/', ObtainAuthToken.as_view(), name='token'),
]
{% endhighlight %}

set up the database,

{% highlight bash %}
$ python manage.py makemigrations
$ python manage.py migrate
$ python manage.py createsuperuser
$ python manage.py runserver
{% endhighlight %}

Now, go to admin and add a new device entry and verify if the tokens are autogenerated and if it did,

{% highlight bash %}
$ http post http://127.0.0.1:8000/token/ mac_address=mac_address 
{
    "token": "bc3b1b99c7a62753428c6169400617c9a539b7b4"
}
# if you don't have httpie, install it by `sudo pip install httpie`
{% endhighlight %}

So, we've setup Token generation for our application, but we haven't done anything for token authentication part,

We have to have our own permission class, here I'm going to use our own IsAuthenticated permission class.
so create a permissions.py file in authtoken app, with the below piece of code.

{% highlight python %}
# authtoken/permissions.py
from rest_framework.permissions import BasePermission

class IsAuthenticated(BasePermission):
    """
    Allows access only to authenticated devices.
    """

    def has_permission(self, request, view):
        try:
        	return request.user and request.user.mac_address
        except:
        	return False
{% endhighlight %}

lets add a view or API endpoint or whatever you call it,

{% highlight python %}
# device/views.py
from django.http import JsonResponse

# DRF
from rest_framework.views import APIView

from authtoken.authentication import TokenAuthentication
from authtoken.permissions import IsAuthenticated

class HelloView(APIView):
    """
        Returns a list of Categories available
    """
    authentication_classes = (TokenAuthentication,)
    permission_classes = (IsAuthenticated,)

    def get(self, request):
        return JsonResponse({'Hello':'View'})
{% endhighlight %}

add our new view to main urls file,

{% highlight python %}
# projectname/urls.py
from django.conf.urls import include, url
from django.contrib import admin
from authtoken.views import ObtainAuthToken
from device.views import HelloView

urlpatterns = [
    url(r'^admin/', include(admin.site.urls)),
    url(r'^token/', ObtainAuthToken.as_view(), name='token'),
    url(r'^helloview/', HelloView.as_view(), name='hello_view'),
]
{% endhighlight %}

Now, try accessing the added url and check it with and without token and see the authentication works

{% highlight bash %}
$ http http://127.0.0.1:8000/helloview/ "Authorization: Token bc3b1b99c7a62753428c6169400617c9a539b7b4"
{
    "Hello": "View"
}
{% endhighlight %}

### What's next!

We could make our token to expire at a certain time delta. For this, we could use [django-rest-framework-expiring-tokens](https://github.com/JamesRitchie/django-rest-framework-expiring-tokens) project directly or bring those features into your own `authtoken` app if you like.
