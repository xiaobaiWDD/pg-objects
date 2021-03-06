# pg-objects

Create PostgreSQL databases, schemas, and roles; grant and revoke privileges.

There is no PyPI package for this as I have released quite a few in the past only to discover
months down the line that perhaps they are not as useful as they seemed to be. Use this at your
own risk.

First blog post on the subject: https://notes.zilupe.com/posts/pg_objects/

## Usage

A working Python example is in ``example.py``.

Command-line interface (CLI): ``python -m pg_objects.cli --help``

CLI can be passed a JSON representation of the object graph and it will create all the 
managed objects for you.

This does not work on Amazon RDS for PostgreSQL because Amazon have bastardised the database
and you don't have a super user. Don't use Amazon RDS for PostgreSQL unless you want to have
a separate cluster for each use case and pay accordingly.

## Story

I have been trying to express PostgreSQL and Redshift permission objects declaratively 
for the last year and a half. This is roughly what I am after: 

```yaml
Objects:

    - Type: User
      Name: u

    - Type: Database
      Name: d
      Owner: u
```

The greatest challenge so far has been to create and drop all the objects in the right
order. Recently I realised that if the object dependencies are expressed 
in a graph then topological sort can be used to calculate the order of the operations.
For create operations we process objects in topological order, and for drop operations we
process them in reverse topological order.

Another important insight to help with organising the code was to express 
the relationships between two objects as another object. For example, the fact that
a `Database:d` is owned by `User:u` is better expressed when behind the scenes
you introduce a separate object `DatabaseOwner:d+u`. The dependencies then are:

```yaml
Dependencies:

    - Object: User:u

    - Object: Database:d
      DependsOn:
        - User:u

    - Object: DatabaseOwner:d+u
      DependsOn:
        - Database:d
        - User:u

```

The separation of database from database ownership allows us to remove the owner of the database
before attempting to drop the database or the user.

The topological order of the vertices of the above graph is:

```yaml
TopologicalOrder:

    - User:u
    - Database:d
    - DatabaseOwner:d+u
```

Another major problem is how privileges such as `ON ALL TABLES` apply only to existing tables. For
tables created later one must `ALTER DEFAULT PRIVILEGES` which require knowing in advance who is going
to create the objects (tables) to which these default privileges apply.

### Permissions Model

By default, the implicit `PUBLIC` group has access to all databases. Default privileges 
are per database, so there is no default privilege we can create to avoid this. For every newly
created database you have to revoke public group's access to the database.
  
    REVOKE ALL PRIVILEGES ON DATABASE {self.name} FROM GROUP PUBLIC

* No grants -- all privileges to be managed by the super user which runs *pg-objects*.
* No user-specific privileges -- all privileges are group-specific
* All referenced groups, users, databases, and schemas should be *managed* -- an object
  is *managed* if the object graph contains an explicit declaration of the object.
* `ALL TABLES` privileges only

##### `NOINHERIT`

Unless specified otherwise, all users are created with `NOINHERIT` which means that they
do not automatically inherit the privileges of groups they belong to. This means that
users are forced to call `SET ROLE groupname` before using privileges associated
with a group. This makes managing default privileges and table ownership easier
as tables aren't created by individual users.

However, it seems that if user has `NOINHERIT`, it is impossible for them to connect
to a database access to which is provided through group membership. **This DOES NOT work**:

```text
postgres=> SET ROLE devops;
postgres=> \c devopsdb
FATAL:  permission denied for database "devopsdb"
DETAIL:  User does not have CONNECT privilege.
```

This forces us to declare `CONNECT` privilege on individual users.
