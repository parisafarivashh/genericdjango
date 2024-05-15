# GenericForeignKey
If you are going to use GenericForeignKey in your current project,

make sure you have a contenttypes framework in your INSTALLED_APPS settings 
because GenericForeignKey depends on it.


```
INSTALLED_APPS = [
    # other installed apps
    'django.contrib.contenttypes', # -> make sure you have this
]
```

If you have gone through the Django source code to figure out how they have implemented the activity log in the admin site,

you will find out that they have used ContentType to achieve that.     
So make our own activity log using ContentType and GenericForeignKey.

Here I have created ActivityLog model to store the activity log (create, update, delete), Profile to have a user profile, and Company where the user can have a company

```commandline
from django.db import models
from django.contrib.auth.models import User
from django.contrib.contenttypes.fields import GenericForeignKey
from django.contrib.contenttypes.models import ContentType


class ActivityLog(models.Model):
    CREATE = 'C'
    UPDATE = 'U'
    DELETE = 'D'
    ACTIVITIES = (
        (CREATE, 'Create'),
        (UPDATE, 'Update'),
        (DELETE, 'Delete'),
    )
    content_type = models.ForeignKey(
        ContentType,
        on_delete=models.CASCADE
    )
    object_id = models.PositiveIntegerField()
    content_object = GenericForeignKey('content_type', 'object_id')
    activity = models.CharField(max_length=1, choices=ACTIVITIES)
    # you can add whatever fields that you want to keep for activity
    # like user, created_at, and so on.
    

class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE)

    
class Company(models.Model):
    owner = models.OneToOneField(Profile, on_delete=models.CASCADE)
```
## in ActivityLog model:

1) we provide a ForeignKey relation to the built-in Django 
   modelContentType, 
   which keeps track of all models in the Django application form INSTALLED_APPS

2) we create a PositiveIntegerField to store the primary key value of 
   whichever model you will reference later on
3) where you specify the GenericForeignKey relation and then pass the names of 
   the two fields created above.  
   If these fields are named “content_type” and 
   “object_id”,  
   you can omit this — those are the default field names 
   GenericForeignKey will look for.

### This is how you create your GenericForeignKey relationship that is not specific to a single model.
### Now you can store the activity as below;

```commandline

profile=Profile.objects.create(user=user)

activity1 = ActivityLog(content_object=profile, activity=ActivityLog.CREATE)
activity1.save()
```

### improve our relationship between models by using GenericRealtion

```
from django.contrib.contenttypes.fields import GenericForeignKey, GenericRelation


class Profile(models.Model):
    # ...
    activity = GenericRelation(ActivityLog)
    

class Company(models.Model):
    # ...
    activity = GenericRelation(ActivityLog)
```

### Now after adding the generic relation we can now create the activity log for the above code as below

```commandline

profile = Profile.objects.create(user=user)
profile.activity.create(activity=ActivityLog.CREATE)

```

### you may create signals in Django to create the activity log whenever the record in the database is created, updated, or deleted. 

```commandline
@receiver(post_save, sender=Profile)
def activity_on_save_for_profile(sender, instance, created, **kwargs):
    if created:
        instance.activity.create(activity=ActivityLog.CREATE)
    else:
        instance.activity.create(activity=ActivityLog.UPDATE)
```

#### Note: 
```
By using the GenericRelation whenever we delete the data of profile and company
our activity log will be deleted for that specific instance.
If you don’t want that behaviour then you souldn’t use the GenericRelation in models.
```

### disadvantage

1) It will add complexity to your code.
2) You cannot use filter() and exclude() with GenericForeignKey as you used 
   to do for ForeignKey.
#### you cannot do this
```
profile = Profile.objects.first()
Address.objects.filter(content_object=profile)
Address.objects.get(content_object=profile)
```
3) It does not accept an on_delete argument to customize the behavior for 
deletions of data.
