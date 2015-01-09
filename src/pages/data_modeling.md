# Data modeling
*Objectives: (a) provide the right way to work with data in slick; (b) introducing more query examples (update and delete)*

Brief overview of chapter objectives.
- Use exercises and examples to define some more of the Schema.
  - start with User

## Rows

<!-- I'm going to ignore HList for the time being as they seem overly complicated and not essential.-->
We model rows of a table using either tuples or case classes. In either case, they contain the types of the columns we wish to expose.  Let's look a simple example of both, we'll define a `user` so we don't need to store their name in the `message` table.

~~~ scala
  final type  TupleUser = (Long,String)

  final case class CaseUser(id:Long,name:String)
~~~

Nothing very magical happening here. There is a little more typing involved in defining the case class, but we get a lot more meaning using case classes.  A little more to be defined in the table definition when using case classes, which we will look at in the next section.

_TODO: Talk about types?_

##Tables

Now we have a row, we can define the table it comes from.  Let's looks at the definition of the `User` table and walk through what is involved.

~~~ scala
  final class TupleUserTable(tag: Tag) extends Table[TupleUser](tag, "user") {
    def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
    def name = column[String]("name")
    def * = (id, name)
  }
  final class UserTable(tag: Tag) extends Table[User](tag, "user") {
    def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
    def name = column[String]("name")
    def * = (id, name) <> (User.tupled,User.unapply)
  }
~~~

We've defined two versions of the the `User` table, one using a tuple, the other a case class.  As you can there is little difference between the two, the kind the `Table` class and the definition of the `*` method, we'll come back to this.

First let's look at how this class relates to the database table.  The name of the table is given as a parameter, in this case the `String` "user" --- `Table[User](tag, "user")`. An optional schema name can also be provided, if required by your database.

Next we define methods for each of the tables columns. These call the method `column` with it's type, name and zero of more options. This is rather self explainitory --- `name` has the type `String` and is mapped to a column `name`. It has no options, we'll explore column in the rest of this chapter.

Finally, we come back to `*`. It is the only method we are required to implement. It defines the default projection for the table.  That is the row object we defined earlier. If we are not using tuples we need to define how Slick will map between our row and projection. We do this using the `<>` operator, supplying two methods --- one to wrap a returned tuple into our type and another to unwrap our type into a tuple. In the `User` example `User.tupled` takes a tuple and returns a User, while `User.unapply` takes a user and returns an `Option` of `(Long,String)`.

<div class="callout callout-info">
**Expose only what you need**

We can hide information by excluding it from our row definition. The default projection controls what is returned and it is driven by our row definition.

</div>

<!--
I think something like this should go here, but meh.
From now on, we will use case classes in our examples as they are easier to reason about.
-->

For the rest of the chapter we'll look at some more indepth areas of data modelling.

##Primary & Foreign keys

There are two methods to declare a column is a primary key.
In the first we declare a column is a primary key using class `O` which provides column options. We have seen examples of this in `Message` and `User`.

~~~ scala
def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
~~~

The second method uses a method `primaryKey` which takes two parameters --- a name and a tuple of columns.
This is useful when defining compound primary keys.  As an example, if we wished to have a primary key of both `id` and `name` on the `User` table, we would do the following:

~~~ scala
def pk = primaryKey("pk_user", (id,name))
~~~

This would produce the following table:

~~~ sql
essential-slick-> \d user
                                 Table "public.user"
 Column |          Type          |                     Modifiers
--------+------------------------+---------------------------------------------------
 id     | bigint                 | not null default nextval('user_id_seq'::regclass)
 name   | character varying(254) | not null
Indexes:
    "pk_user" PRIMARY KEY, btree (id, name)
 ~~~

Foreign keys are declared in a similar manner to compound primary keys, with the method --- `foreignKey`. `foreignKey` takes four required parameters:
   * a name;
   * the column(s) that make the foreignKey;
   * the `TableQuery`that the foreign key belongs to, and
   * a function on the supplied `TableQuery[T]` taking the supplied column(s) as parameters and returning an instance of `T`.

As an example let's improve our model by replacing the `from` column in `Message` with a foreign key to the `User` primary key.

~~~ scala


  final case class User(id: Long, name: String)

  final class UserTable(tag: Tag) extends Table[User](tag, "user") {
    def id = column[Long]("id",O.PrimaryKey,O.AutoInc)
    def name = column[String]("name")
    def * = (id, name) <> (User.tupled, User.unapply)

  }

  lazy val users = TableQuery[UserTable]

  final case class Message(id: Long, from: Long, content: String, when: DateTime)

  final class MessageTable(tag: Tag) extends Table[Message](tag, "message") {
    def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
    def fromId = column[Long]("from")
    def from = foreignKey("from_fk", fromId, users)(_.id)
    def content = column[String]("content")
    def when = column[DateTime]("when")
    def * = (id, fromId, content, when) <> (Message.tupled, Message.unapply)
  }

  lazy val messages = TableQuery[MessageTable]

~~~

This will produce the following table:

~~~ sql
essential-slick=> \d user;
                                 Table "public.user"
 Column |          Type          |                     Modifiers
--------+------------------------+---------------------------------------------------
 id     | bigint                 | not null default nextval('user_id_seq'::regclass)
 name   | character varying(254) | not null
Indexes:
    "user_pkey" PRIMARY KEY, btree (id)
Referenced by:
    TABLE "message" CONSTRAINT "from_fk" FOREIGN KEY ("from") REFERENCES "user"(id)
~~~

<div class="callout callout-info">
**Slick isn't an ORM **

Adding foreign keys to our data model does not mean we can traverse from `Message` to `User`, as Slick is not an ORM.

We can however compose our queries and join the return the `User` we are interested in.

</div>

_TODO: Add a query example_

##Value classes

_TODO use a value class to define message content._


##Null columns

Thus far we have only looked at non null columns, however sometimes we will wish to modal optional data. Slick handles this in an idiomatic scala fashion using `Option[T]`. Let's expand our data model to allow direct messaging by adding the ability to define a recipient to `Message`:

~~~ scala

  final case class Message(id: Long, from: Long, to: Option[Long], content: String, when: DateTime)

  final class MessageTable(tag: Tag) extends Table[Message](tag, "message") {

    def id = column[Long]("id", O.PrimaryKey, O.AutoInc)
    def fromId = column[Long]("from")
    def from = foreignKey("from_fk", fromId, users)(_.id)
    def toId = column[Option[Long]]("to")
    def to = foreignKey("to_fk", toId, users)(_.id)
    def content = column[String]("content")
    def when = column[DateTime]("when")

    def * = (id, fromId, toId, content, when) <> (Message.tupled, Message.unapply)
  }

~~~


<div class="callout callout-warn">
***Equality***

While nullability is treated in an idiomatic Scala fashion using `Option[T]`. It behaves differently.



</div>


##Row and column control (autoinc etc)




##Custom types & mapping

  - explain `when` in Message


##Example using date and time?

##Virtual columns and server-side casts here?

##Exercises