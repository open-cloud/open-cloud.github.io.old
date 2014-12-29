---
layout: page
title: Developer Guide
---

##Development Environment

##Testing Framework

##Reporting Bugs

##Adding Views

##Adding Services

##Data Model Conventions

###Modeling Basics

We first establish Django terminology, and how it relates to common
object-related terminology. An Object Class is called a Model in
Django. A Django Model consists of a set of Fields (equivalent to
saying an Object Class is defined by a set of Attributes). Each Django
Field has a Type and each Type has a set of Attributes. Some of these
Attributes are core (common across all Types) and some are
Type-specific. Relationships between Models are expressed by Fields
with one of a set of distinguished relationship-oriented Types (e.g,
OneToOneField). Finally, an Object is an instance (instantiation of) a
Model, where each Object has a unique primary key (or more precisely,
a primary index into the table that implements the Model). By
convention, that index/key should be the Django AutoField id which is
automatically created for any Model that has not identified a separate
unique primary key. The Django default primary key is always “id” for
system level tables, and “pk” for model tables.

####Field Types

There are various built in Fields that may be specified at Class
declaration. The basic field types commonly in use with OpenCloud:

| FieldType          | Why to Use It?     |
|--------------------|--------------------|
| BooleanField       | Displays as a checkbox in GUI, limits input to valid possibilities.  If None is a valid option, please use **NullBooleanField** which uses default widget of choice None, Yes, No.|
| CharField          | Allows for String representation, additional required attributes are “max_length”. This should be used for small to large size strings. For mulitiple sentences, please use TextField.|
| DateTimeField      | Special field that maps to python DateTime.  May set optional attributes of *auto_now* (update field each time it is saved), *auto_add_now* (update field when created only) |
| EmailField         | Automatically checks for well formed Emails based on RFC3696/5321. Note that for full compliance we need to set the length to 254[a], default is 75. |
| FloatField         | Used to represent a real number, default widget is TextInput.|
| IntegerField       | Used to represent Integer values, default widget is TextInput[b].|
| GenericIPAddressField | Able to validate ipv4 and ipv6 flavors of addresses, default widget is TextInput.|
| URLField           | Verifies well formed URL, max_length is defaulted to 200.  GUI convenience is that the value of the field is displayed as a clickable link above the input widget.|
| SlugField          | Used to represent a short label for something, containing only letters, numbers, underscores or hyphens.  This should be used for the name of Tags so that there is a better chance of promoting the Tag to be an actual field attribute once the model has been solidified.|
	
####Core Field Attributes

List of Field level optional attributes currently in use:

| Attribute          | Affect             |
|--------------------|--------------------|
| verbose_name=”Some long format” | Provides a specific label to be used in the GUI display for this particular field. This differs from the field name itself which is used to create the column in the database table.|
| verbose_name_plural=”Some long formats” | Way to override the verbose name for this field.|
| help_text=”Help with this specific field” | Provides some context based help for the specific field, this will show up in the GUI display.|
| default=”Some appropriate default” | Allows a predefined default value to be specified.|
| choices=CHOICES_TUPLE_LIST | Allows the field to be filled in from an enumerated choice.  Example: ROLE_CHOICES = (('admin', 'Admin'), ('pi', 'Principle Investigator'), ('user','User'))|
| unique=True |	Requires that the field be unique across all entries.|
| blank=True | Allows the field to be present but empty.|
| null=True | Allows the field to have a value of null if the field is blank.|
| editable=False | If you would like to make this field a readOnly field to the user.|
	 
List of Field level optional attributes we should not use, or use judiciously:

| Attribute          | Why                |
|--------------------|--------------------|
| primary_key        | Some of the plugins we are making use of particularly in the REST area do not do well with CharField’s as primary_key.  In general, it is best to use the system primary key instead, and put a db_index=True, unique=True on the CharField you would have used.|
| db_column, db_tablespace | Convention is to use the Field name as the db column, and use verbose_name if you want to change the display.  For tablespace, all models should be defined within the application they are specified in - overwriting the tablespace will make it more challenging for the next developer to find and fix any issues that might arise.|

List of Field level optional attributes we will likely need/want to
use:

| Attribute          | Affect             |
|--------------------|--------------------|
| db_index=True      | Allow for quicker queries if you are going to order by, or filter by the specified field.|
| error_messages={ } | Pass in a dictionary of error keys, and the message you want to display if the error occurs.  For example: null, blank, invalid, invalid_choice, and unique  Other error messages may be present per FieldType.|
| validators=[]      | Allows a list of validators to be run before committing the value of this field to the model
(https://docs.djangoproject.com/en/dev/ref/validators/).|

####Expressing Relationships

There are a few different types of Relationship based fields offered
in Django.

| FieldType          | When to Use it?    |
|--------------------|--------------------|
| ForeignKeyField    | Used to represent a 1 to Many relationship. For example: Sliver’s may have 1 and only 1 Node.  Node’s may have 0 or more Slivers. Can also be used to represent recursive relationships for the same object types by providing ‘self’ as the relationship (first position) parameter.|
| ManyToManyField    | Used to represent an N to N relationship. For example:
Deployment may have 0 or more Sites Sites may have 0 or more Deployments.|
| OneToOneField      | Not currently in use, but would be useful for applications that wanted to augment a core class with their own additional settings.  Possibly Syndicate or RequestRouter later. This has the same affect as a ForeignKey with unique=True.  The difference is that the “reverse” side of the relationship will always be 1 object (not a list).|
| GenericForeignKey[c] | Not currently in use, but can be used to specify a non specific relation to “another object”. Meaning object A relates to any other object.  This relationship requires a reverse attribute in the “other” object to see the relationship - but would primarily be accessed through the GenericForeignKey owner Model. Example: https://docs.djangoproject.com/en/dev/ref/contrib/contenttypes/#id1. The nuances/complexities of each of these relationships is brought about by the additional optional attributes that can be ascribed to each Field.|

####Optional Attribute Side Effects

| Attribute          | Key Type           | Why                |
|--------------------|--------------------|--------------------|
| limit_choices_to   | ForeignKeyField, ManyToManyField | Not currently in use. Good to limit the possible selection choices for a remote relationship by some constraint.  Constraints may either be expressed by a dictionary of lookups, or by a Q object. (See https://docs.djangoproject.com/en/dev/topics/db/queries/#complex-lookups-with-q.) Example of data limit for another object: limit_choices_to = {'pub_date__lte': datetime.date.today}|
| relate_name        | ForeignKeyField, ManyToManyField | The name to use for the relation back to this one.  This is extremely useful both to provide additional context in the way the objects are related but also when you have the same objects and object types related in different contexts to the same instance.  It allows for lookups and incorporation into Admin Forms based on the relationship. Example: SliceMembership: user = models.ForeignKey('User', related_name='slice_memberships') slice = models.ForeignKey('Slice', related_name='slice_memberships'). ‘+’ is a special meaning in this field and if the only character, or the trailing character of the field; will have the effect of eclipsing the backward relationship so that it is unseen by the related object.|
| on_delete          | ForeignKeyField | Default behavior is to behave like “ON DELETE CASCADE”.  This may be turned of so long as the “null=True, blank=True” options are also set. Example: user = models.ForeignKey(User, blank=True, null=True, on_delete=models.SET_NULL)|
| symmetrical        | ManyToManyField | Only used when ManyToManyField definition is against “self”. class Person: friends = models.ManyToManyField(“self”). Would then result in if Person Alice is friended to Person Brian, the Brian is also Friends with Alice.|
| through            | ManyToManyField | Django automatically creates a mapping table for ManyToMany relationships. By using this field, you can specify the exact mapping table you would like to use, therefore allowing you to augment/extend that table with additional data. See example[d]: https://docs.djangoproject.com/en/dev/topics/db/models/#intermediary-manytomany.|
| parent_link        | OneToOneField | Used only for multiple inheritance (specifically from a concrete class) to avoid additional/extra OneToOneFields being created via subclassing. Django documentation for Related Fields can be found at:
https://docs.djangoproject.com/en/dev/ref/models/fields/#module-django.db.models.fields.related.|

####Avoid List

Avoid using the following optional attributes, they can have adverse
effects on data integrity and REST relationships:

| Attribute          | Affect             |
|--------------------|--------------------|
| to_field           | Would allow the relationship to exist against a field other than the primary key in the remote object.|
| db_constraint      | Controls whether or not a constraint should be created in the database for the foreignKey relationship. It defaults to True, if you set it to False you could cause database pointers to non-existent data. We should avoid this.|
