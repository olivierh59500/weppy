Migrations
==========

*New in version 0.6*

Migrations are the basic instrument needed to propagate changes you make to your models into your database schema.

Under default behavior the weppy `DAL` class will be initialized with the automatic migrations feature of the *[pydal](https://github.com/web2py/pydal)* library enabled: this is quite handy for small applications that won't frequently change the structure of their models, since it will propagate the changes of the models to the database tables as soon as you load your application.    
On the other hand, if you have an application that evolves a lot and requires a lot of changes on the models, or when you need some control on the migration process for the production side, this feature may produce unwanted side effects, since it stores the mgiration status in some files under the *databases* directory of your application and migrations are performed on the database every time you push new code in your models.

This is why weppy comes with a migration engine based on *revisions*: this will use migration files containing the instructions to be performed on the database side and will store the current migration status on the database itself, fact that prevents inconsistencies on the migration status of your application if you are running the code from several machines.

> **Note:** we **strongly reccomend** you to turn off automatic migrations for every application that will be run on production side. The automatic migrations and the ones performed by the migration engine have some slight differences; while we will document operations supported by the second system, the detection performed by the automatic one depends on the *pydal* library. If you need more informations about this you should check the [web2py docs](http://www.web2py.com/books/default/chapter/29/06/the-database-abstraction-layer#Migrations).

The first step that has to be performed in order to use the migration engine is to turn off the automatic migrations with the `auto_migrate` parameter of the `DAL` class:

```python
db = DAL(app, auto_migrate=False)
```

this will, indeed, prevent the `DAL` class to automatic migrate your database if you change your models.

Then, you will need to use the migration commands integated with weppy in order to generate, apply and revert migrations on your database. In order to avoid you the pain of writing a lot of migration code aside with your models, weppy will automatically generate the migration scripts for you starting from your models' code.    
In the next sections we will describe all of this using the *bloggy* application we saw in the [tutorial chapter](././tutorial) as an example.

Generating your first migration
-------------------------------

Once you have written your models, you will need to create the relevant tables on the database to start performing operations.

In the bloggy example we defined three models, and since we turned off automatic migrations, we need to create a migration that will add the relevant tables to the database. This can be quite immediate if you use the weppy generation command:

```
$ weppy -a bloggy.py migrations generate -m "First migration"
> Generated migration for revision 4ceb82ecd8e4
```

As you can see the generate command accepts a `-m` option which will be the message for the migration and also its name, and prints out the revision number for the generated migration. This number is unique and will identify the single migration.

Now if you look into the *migrations* folder of the bloggy application, you will find a *4ceb82ecd8e4\_first\_migration.py* file, containing all the operations needed to create the tables defined by our models. Since we used the `Auth` module in bloggy, beside the tables for the `User`, `Post` and `Comment` models we will find also the operations that will create the ones for the authorization module as well:

```python
"""First migration

Migration ID: 4ceb82ecd8e4
Revises:
Creation Date: 2016-01-23 17:29:38.642478
"""

from weppy.dal import migrations


class Migration(migrations.Migration):
    revision = '4ceb82ecd8e4'
    revises = None

    def up(self):
        self.create_table(
            'users',
            migrations.Column('id', 'id'),
            migrations.Column('created_at', 'datetime'),
            migrations.Column('updated_at', 'datetime'),
            migrations.Column('email', 'string', length=255),
            migrations.Column('password', 'password', length=512),
            migrations.Column('registration_key', 'string', default='', length=512),
            migrations.Column('reset_password_key', 'string', default='', length=512),
            migrations.Column('registration_id', 'string', default='', length=512),
            migrations.Column('first_name', 'string', notnull=True, length=128),
            migrations.Column('last_name', 'string', notnull=True, length=128))
        self.create_table(
            'auth_groups',
            migrations.Column('id', 'id'),
            migrations.Column('created_at', 'datetime'),
            migrations.Column('updated_at', 'datetime'),
            migrations.Column('role', 'string', default='', length=255),
            migrations.Column('description', 'text'))
        self.create_table(
            'auth_memberships',
            migrations.Column('id', 'id'),
            migrations.Column('created_at', 'datetime'),
            migrations.Column('updated_at', 'datetime'),
            migrations.Column('user', 'reference users', ondelete='CASCADE'),
            migrations.Column('authgroup', 'reference auth_groups', ondelete='CASCADE'))
        self.create_table(
            'auth_permissions',
            migrations.Column('id', 'id'),
            migrations.Column('created_at', 'datetime'),
            migrations.Column('updated_at', 'datetime'),
            migrations.Column('name', 'string', default='default', notnull=True, length=512),
            migrations.Column('table_name', 'string', length=512),
            migrations.Column('record_id', 'integer', default=0),
            migrations.Column('authgroup', 'reference auth_groups', ondelete='CASCADE'))
        self.create_table(
            'auth_events',
            migrations.Column('id', 'id'),
            migrations.Column('created_at', 'datetime'),
            migrations.Column('updated_at', 'datetime'),
            migrations.Column('client_ip', 'string', length=512),
            migrations.Column('origin', 'string', default='auth', notnull=True, length=512),
            migrations.Column('description', 'text', default='', notnull=True),
            migrations.Column('user', 'reference users', ondelete='CASCADE'))
        self.create_table(
            'posts',
            migrations.Column('id', 'id'),
            migrations.Column('title', 'string', length=512),
            migrations.Column('text', 'text'),
            migrations.Column('date', 'datetime'),
            migrations.Column('user', 'reference users', ondelete='CASCADE'))
        self.create_table(
            'comments',
            migrations.Column('id', 'id'),
            migrations.Column('text', 'text'),
            migrations.Column('date', 'datetime'),
            migrations.Column('user', 'reference users', ondelete='CASCADE'),
            migrations.Column('post', 'reference posts', ondelete='CASCADE'))

    def down(self):
        self.drop_table('comments')
        self.drop_table('posts')
        self.drop_table('auth_events')
        self.drop_table('auth_permissions')
        self.drop_table('auth_memberships')
        self.drop_table('auth_groups')
        self.drop_table('users')
```

As you can see, every migration file contains a `Migration` class that has the `revision` attribute, used to indentify the revision number of the migration, and the `revises` attribute, that in our case is `None`. This attribute will specify the prior revision used to generate the migration: since we just created the first migration for the application, we don't have any other migration to use as a base.

The `Migration` class also has two methods, the `up` and `down` ones, that will specify the operations that should be run when we apply the migration and the ones that should be performed when we want to rollback the migration changes and return to the previous state. In this case, we have seven `create_table` operations for the `up` section, and the seven *opposite* `drop_table` operations in the `down` ones.

`create_table` and `drop_table` are directives of the weppy migration engine. We will see that the `Migration` class have several methods like these that can be used to alter tables and columns in your database.

Running your fist migration
---------------------------

Once we generated our first migration, we need to run it in order to apply the canges to the database. We can use the *up* command of weppy migrations:

```
$ weppy -a bloggy.py migrations up
> Performing upgrades against sqlite://dummy.db
> Performing upgrade: <base> -> 4ceb82ecd8e4 (head), First migration
> Adding revision 4ceb82ecd8e4 to schema
> Succesfully upgraded to revision 4ceb82ecd8e4: First migration
```

As you can see, the command prints out some information regarding the operations it runs: it says on which database the operations are performed, which migrations are used to upgrade the database and which revision is stored on the schema. In this case the migration was performed successfully, in fact we can check the current revision of the database:

```
$ weppy -a bloggy.py migrations status
> Current revision(s) for sqlite://dummy.db
4ceb82ecd8e4 (head)
```

If somehow something were wrong with the migration, the up command would notifiy it and rollback the migration, and the *status* command would displayed a different revision.

Running a second migration
--------------------------

Whenever you make changes to the application models, you need to create a migration that will modify your database accordingly. Considering the *bloggy* example again, if we write down another model:

```python
class Tag(Model):
    name = Field()
```

and add it to our `DAL` instance:

```python
db.define_models(Post, Comment, Tag)
```

we can run the *generate* command again:

```
$ weppy -a bloggy.py migrations generate -m "Add tags"
> Generated migration for revision 4dee31071bf8
```

and we will find a *4dee31071bf8\_add\_tags.py* file in the *migrations* folder:

```python
"""Add tags

Migration ID: 4dee31071bf8
Revises: 4ceb82ecd8e4
Creation Date: 2016-01-24 14:43:16.963860

"""

from weppy.dal import migrations


class Migration(migrations.Migration):
    revision = '4dee31071bf8'
    revises = '4ceb82ecd8e4'

    def up(self):
        self.create_table(
            'tags',
            migrations.Column('id', 'id'),
            migrations.Column('name', 'string', length=512))

    def down(self):
        self.drop_table('tags')
```

As you can see the engine found that the change to perform is just the one of creating the *tags* table. Now we can upgrade our database to this revision as we did for the first migration:

```
$ weppy -a bloggy.py migrations up
> Performing upgrades against sqlite://dummy.db
> Performing upgrade: 4ceb82ecd8e4 -> 4dee31071bf8 (head), Add tags
> Updating schema revision from 4ceb82ecd8e4 to 4dee31071bf8
> Succesfully upgraded to revision 4dee31071bf8: Add tags
```

As we increase the number of migrations for our application, the *history* command can be useful to check out them as an ordered list:

```
$ weppy -a bloggy.py migrations history
> Migrations history
4ceb82ecd8e4 -> 4dee31071bf8 (head), Add tags
<base> -> 4ceb82ecd8e4, First migration
```

where the most recent is on top and the last one is the oldest one.

Downgrading migrations
----------------------

Whenever you need to rollback to a previous revision of your schema, you can use the *down* command:

```
$ weppy -a bloggy.py migrations down -r base
> Performing downgrades against sqlite://dummy.db
> Performing downgrade: 4ceb82ecd8e4 -> 4dee31071bf8 (head), Add tags
> Updating schema revision from 4dee31071bf8 to 4ceb82ecd8e4
> Succesfully downgraded from revision 4dee31071bf8: Add tags
> Performing downgrade: <base> -> 4ceb82ecd8e4, First migration
> Removing revision 4ceb82ecd8e4 from schema
> Succesfully downgraded from revision 4ceb82ecd8e4: First migration
```

The `-r` option is required and has to be the revision to be downgraded. Whenever you specify a revision identifier that is not the current *head* state, weppy will downgrade every revision applied after the revision you specified and the one you specified too.    
When this parameter is set to `base`, weppy will use the first migration in history: in our case we returned the database back to the beginning. In fact, if we run the status command:

```
$ weppy -a bloggy.py migrations status
> Current revision(s) for sqlite://dummy.db
No revision state found on the schema.
```

we can see that no revision is loaded on it.

Generation and changes detection
--------------------------------

Unfortunately, the migration generator shipped with weppy is not able to detect all the changes you might make on the models. Here we list what the generator can detect and what it cannot, but, please, consider that this list may change on new releases.

The generator detects:

- tables additions, removals
- columns additions, removals
- type, nullable and default changes on columns

While it won't detect:

- tables renaming
- columns renaming
- unique constraints changes on columns

The changes unsupported by the generation should be performed using the appropriate operations when available or writing down the SQL code manually and using the `db._adapter.execute()` command.

Empty migrations and operations
-------------------------------

Aside with the generated migrations, you can obviously generate new empty migration files and write down the operations on your own. In fact, if you run the *new* command:

```
$ weppy -a bloggy.py migrations new -m "Custom migration"
> Created new migration with revision 57f23e051fa3
```

you will find out a *57f23e051fa3\_custom\_migration.py* file inside your *migrations* folder:

```python
"""Custom migration

Migration ID: 57f23e051fa3
Revises: 4dee31071bf8
Creation Date: 2016-01-24 19:17:24.773448

"""

from weppy.dal import migrations


class Migration(migrations.Migration):
    revision = '57f23e051fa3'
    revises = '4dee31071bf8'

    def up(self):
        pass

    def down(self):
        pass
```

As you can see is just like the generated migrations, excepting the fact that the `up` and `down` methods are empty. You can now write down your migration code, using the first method to declare the changes you need to perform, and the second one to define the operations needed to rollback to the current state. The next paragrahs describe the instance methods available on the `Migration` class in order to migrate your database structure.

### create\_table

The `create_table` method will, indeed, create a new table in your database. It accepts the name of the table as the first parameter and a list of columns:

```python
self.create_table(
    'tags',
    migrations.Column('id', 'id'),
    migrations.Column('name', 'string', length=512))
```

The `Column` class behaves quite like the `Field` one, except that it has the column name as first parameter. Also the types are quite different, since you will have *boolean* instead of *bool* and *integer* in place of *int*. The first column of a table should always be the one with name *id* and type *id*, since it's used by weppy in order to perform operations on the tables.

### drop\_table

The `drop_table` method is the opposite of the `create_table` one, since it will permanently remove a table and all its data from your database. It accepts just one parameter: the name of table to drop.

```python
self.drop_table('tags')
```

### add\_column

The `add_column` method will add an additional column to the involved table. Requires the name of the table as first parameter and a `Column` object as second argument.

```python
self.add_column('tags', migrations.Column('count', 'integer', default=0))
```

### drop\_column

The `drop_column` method will remove a column from the specified table. Needs two parameters: the name of the involved table and the name of the column to remove.

```python
self.drop_column('tags', 'count')
```

### alter\_column

The `alter_column` method allows you to make changes on a specific column. It needs the name of the table as first parameter, the name of the column as second parameter and then named parameters for the changes. 

The supported changes are on type, default and nullable option. You can combine any of all these three options in a single `alter_column` command, but you should add to the command the existing values for the other options:

```python
#: change notnull from True to False
self.alter_column('things', 'name',
    existing_type='string',
    notnull=False)
#: remove default value
self.alter_column('things', 'value',
    existing_type='float',
    default=None,
    existing_notnull=False)
#: change type from 'string' to 'integer'
self.alter_column('things', 'count',
    existing_type='string',
    type='integer',
    existing_notnull=False)
```

Just as a recap, here is the list of named arguments supported by the `alter_column` operation (all of them are set to `None` by default):

- `notnull`
- `default`
- `type`
- `existing_notnull`
- `existing_default`
- `existing_type`

Custom operations
-----------------

Sometimes you will need to access the database during the migration and run custom operation on it. This is a supported behavior, since the `Migration` class have a `db` attribute which is actually your database, and all the normal operations are supported:

```python
def up(self):
    self.alter_column('things', 'value',
        default=5,
        existing_type='integer',
        existing_notnull=True)
    db(db.things.value == 5).update(value=3)
```

As you can se we added a custom update command inside the upgrade function; and obviously you can add operations to the weppy generated migrations too.

DBMS support
------------

The weppy migration engine is designed to support all the DBMS supported by pyDAL. Still, due to limitations in some engines, some operations are partially or totally not supported. In this section we will list these limitations for every DBMS.

### SQLite

The SQLite engine doesn't support *ALTER TABLE* operations that change the columns. As a direct consequence, any `alter_column` operation you will try to run from a migration will fail.