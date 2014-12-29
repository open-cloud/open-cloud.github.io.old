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
| BooleanField       | Displays as a checkbox in GUI, limits input to valid possibilities.  If None is a valid option, please use NullBooleanField which uses default widget of choice None, Yes, No.|
