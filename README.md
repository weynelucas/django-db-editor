# Django DB adapter

[![Test](https://github.com/weynelucas/django-db-adapter/actions/workflows/test.yml/badge.svg)](https://github.com/weynelucas/django-db-adapter/actions/workflows/test.yml)
[![Package](https://github.com/weynelucas/django-db-adapter/actions/workflows/deploy.yml/badge.svg)](https://github.com/weynelucas/django-db-adapter/actions/workflows/deploy.yml)
[![codecov](https://codecov.io/gh/weynelucas/django-db-adapter/branch/master/graph/badge.svg?token=EZyTLmsPhm)](https://codecov.io/gh/weynelucas/django-db-adapter)
[![PyPI - Release](https://img.shields.io/pypi/v/django-db-adapter.svg)](https://pypi.python.org/pypi/django-db-adapter)
[![PyPI - Python Version](https://img.shields.io/pypi/pyversions/django-db-adapter)](https://pypi.python.org/pypi/django-db-adapter)
[![PyPI - Django Version](https://img.shields.io/pypi/djversions/django-db-adapter)](https://pypi.python.org/pypi/django-db-adapter)
[![Downloads](https://pepy.tech/badge/django-db-adapter)](https://pepy.tech/project/django-db-adapter)
[![Code style: black](https://img.shields.io/badge/code%20style-black-000000.svg)](https://github.com/psf/black)


A flexible toolkit for customize how Django creates the databse objects for the
application schema

# Overview
Django DB adapter is a flexible schema editor backend built to solve the following problems:

- Generate SQL statements for projects working on database-first approach
- All objects created (including created from Django) must have a particular name pattern, like add prefixes and suffixes
- All table columns should be commented
- Inline constraints (primary/foreign keys, unique/check constraints and indexes) are not allowed
- The database user of application is not the owner of the objects and has no privilege to create/alter/drop them (`python manage.py migrate` will not work for this user). All DDL statements generated should include a grant of manipulation privileges (select/insert/update/delete) on created objects for this user/role
- The order of SQL statements matters


# Requirements
- Python (3.6, 3.7, 3.8, 3.9)
- Django (1.11, 2.2)


We highly recommend and only officially support the latest patch release of each Python and Django series.

# Installation
Install using `pip`...

```bash
pip install django-db-adapter
```

Add `'db_adapter'` to your `INSTALLED_APPS` setting.

```python
INSTALLED_APPS = [
    ...
    'db_adapter',
]
```


# Quick Example
Let's take a look at a quick example of using DB adapter to customize the DDL
statements generated by Django.

This example model defines a `Person`, which has a `first_name` and `last_name`:

```python
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30, help_text="It's your last name")

    class Meta:
        db_table = 'person'
```

Add the following to your `settings.py` module:

```python
INSTALLED_APPS = [
    ...  # Make sure to include the default installed apps here.
    'db_adapter',
]

DATABASES = {
    'default': {
        # Make sure to include `db_adapter.db.backends.oracle` as database
        # engine for schema customization
        'ENGINE': 'db_adapter.db.backends.oracle',
        'NAME': 'xe',
        'USER': 'a_user',
        'PASSWORD': 'a_password',
        'HOST': 'dbprod01ned.mycompany.com',
        'PORT': '1540',
    }
}

DB_ADAPTER = {
    'DEFAULT_ROLE_NAME': 'rl_example',
    # Apply this pattern for all tables
    'DEFAULT_DB_TABLE_PATTERN': '"example"."tb_{table_name}"',
    # Ignore some patterns from normalization
    'IGNORE_DB_TABLE_PATTERNS': [
        '"{}"."{}"', # Tables with already declared namespace
        'django_migrations', # Django migrations table
    ],
    'DEFAULT_OBJECT_NAME_PATTERNS': {
        'SEQUENCE': 'sq_{table_name}',
        'TRIGGER': 'tg_{table_name}_b',
        'INDEX': 'ix_{name}',
        'PRIMARY_KEY': 'cp_{name}',
        'FOREIGN_KEY': 'ce_{name}',
        'UNIQUE': 'ct_{name}_uq',
        'CHECK': 'ct_{name}{qualifier}',
    },
    'SQL_FORMAT_OPTIONS': {
        'unquote': True,
        'identifier_case': 'lower',
        'keyword_case': 'lower',
    },
    # Order of SQL statements
    'SQL_STATEMENTS_ORDER': [
        'PRIMARY_KEY',
        'UNIQUE',
        'FOREIGN_KEY',
        'CHECK',
        'INDEX',
        'COMMENT',
        'CONTROL', # Grant/revoke table privileges for specified role (if exists)
        'AUTOINCREMENT', # Sequence and triggers for auto-incremented fields
    ]
}
```

The above `Person` model would create a database table like this:

```sql
create table example.tb_person (
    id number(11),
    first_name nvarchar2(30),
    last_name nvarchar2(30)
);
/

alter table example.tb_person
    add constraint cp_person_id
    primary key (id);
/

alter table example.tb_person
    add constraint ct_person_id_nn
    check (id is not null);
/

alter table example.tb_person
    add constraint ct_person_first_name_nn
    check (first_name is not null);
/

alter table example.tb_person
    add constraint ct_person_last_name_nn
    check (last_name is not null);
/

comment on column example.tb_person.last_name
    is 'It''s your last name';
/

grant select, insert, update, delete
    on example.tb_person
    to rl_example;
/

create sequence example.sq_person
    minvalue 1
    maxvalue 99999999999
    start with 1
    increment by 1
    cache 20;
/

grant select
    on example.sq_person
    to rl_example;
/

create or replace trigger example.tg_person_b
before insert on example.tb_person
for each row
when (new.id is null)
    begin
        select example.sq_person.nextval
        into :new.id from dual;
    end;
/
```

# Release notes

- `v1.0.0` - Apr 16, 2018 - First release
- `v1.0.1` - Apr 16, 2018 - Rename package and fix setup issues
- `v1.0.2` - Apr 17, 2018 - Fix documentation preview
- `v2.0.0` - Mar 1, 2021 - Recreate the entire schema editor backend with more flexible features
- `v2.0.1` - Mar 22, 2021 - Escape single quotes on column comments
