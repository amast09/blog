+++
date = "2018-01-21T11:28:35-05:00"
title = "Scala Database Normalization using Slick"
description = "Use Slick to work with normalized database data in Scala"
tags = [ "Scala", "Slick", "Database" ]
categories = [ "Development", "Scala", "How To" ]
+++

When designing data schemas it is very important to normalize many to many data relationships.

In this blog post I am going to go over how to use slick to work with many to many data relationships within Scala code.

I will assume that you have sbt 0.13.13 or later installed on your machine.

First we will create a scala hello world project to get up and running quickly,

```
$ sbt new sbt/scala-seed.g8
....
Minimum Scala build.

name [My Something Project]: many2many

Template applied in ./many2many
```


To run the newly created scala app run the following commands,

```bash
cd many2many
sbt run
```

The next thing we will do is add Slick and other required dependencies to our project.

Replace the contents of `build.sbt` with the following code,

```
lazy val root = (project in file(".")).
  settings(
    inThisBuild(List(
      organization := "com.example",
      scalaVersion := "2.12.3",
      version      := "0.1.0-SNAPSHOT"
    )),
    name := "Hello",
    libraryDependencies ++= Seq(
      "com.h2database" % "h2" % "1.4.192",
      "com.typesafe.slick" %% "slick" % "3.2.1",
      "org.slf4j" % "slf4j-nop" % "1.6.4",
      "com.typesafe.slick" %% "slick-hikaricp" % "3.2.1",
      "com.typesafe" % "config" % "1.3.1"
    )
  )
```

You should also delete the auto created `test` directory.

After this, we will create the following file, `src/main/resources/application.conf` with the following contents,

```
h2mem1 = {
  url = "jdbc:h2:mem:test1"
  driver = org.h2.Driver
  connectionPool = disabled
  keepAliveConnection = true
}
```

This will be configure our database to be an in-memory database. This way we do not have to spin up and configure an
instance of a postgresql or mysql database.

The many to many data relationship we will model will be books to authors.

* An author can have many books
* A book can have many authors.

Now that we have our slick dependencies defined we can create our scala model case classes and Slick table row classes.

```scala
// Simple Scala data model case classes
case class Author(id: Int, name: String)
case class Book(id: Int, name: String)
```

Then add within our Hello example object the slick Table row classes, 

```scala
  // Slick table row classes
  class Authors(tag: Tag) extends Table[Author](tag, "author") {
    def id = column[Int]("id", O.PrimaryKey)
    def name = column[String]("name")
    def * = (id, name) <> ((Author.apply _).tupled, Author.unapply)
  }
  val authors = TableQuery[Authors]

  class Books(tag: Tag) extends Table[Book](tag, "author") {
    def id = column[Int]("id", O.PrimaryKey)
    def name = column[String]("name")
    def * = (id, name) <> ((Book.apply _).tupled, Book.unapply)
  }
  val books = TableQuery[Books]
```

We now have the ability to insert and query books and authors from our in memory database, lets try it out.

Add the following lines of code below the table row classes,

```scala
// Create a connection to our in-memory database
  val db = Database.forConfig("h2mem1")

  val stephenKing = Author(1, "Stephen King")
  val jkRowling = Author(2, "J. K. Rowling")
  val jrrTolkien = Author(3, "J. R. R. Tolkien")
  val danBrown = Author(4, "Dan Brown")

  val theShining = Book(1, "The Shining")
  val harryPotter = Book(2, "Harry Potter")
  val theLordOfTheRings = Book(3, "Lord of the Rings")
  val theDaVinciCode = Book(4, "The Da Vinci Code")
  val fictiousBook1 = Book(5, "Ficticious Book 1")
  val fictiousBook2 = Book(6, "Ficticious Book 2")

  try {
    val setup = DBIO.seq(
      // Create the tables
      (authors.schema ++ books.schema).create,

      // Insert some authors
      authors += stephenKing,
      authors += jkRowling,
      authors += jrrTolkien,
      authors += danBrown,

      // Insert some books
      books += theShining,
      books += harryPotter,
      books += theLordOfTheRings,
      books += theDaVinciCode,
      books += fictiousBook1,
      books += fictiousBook2,
    )

    val setupFuture = db.run(setup)

    val futureResult = setupFuture.flatMap { _ =>
      // Read all authors and print them to the console
      println("Authors:")
      db.run(authors.result).map(_.foreach(println))
    }.flatMap { _ =>
      // Read all books and print them to the console
      println("Books:")
      db.run(books.result).map(_.foreach(println))
    }

    Await.result(futureResult, Duration.Inf)
  } finally db.close
```

Fairly straight forward code for working with some simple data being stored in our relational in memory database.

Next up, our many-to-many relationship.

We can normalize many-to-many data relationships with an associative table that contains an `author_id` foreign key and a 
`book_id` foreign key, together representing a composite primary key.

Here is the simple model class,

```scala
case class AuthorBook(authorId: Int, bookId: Int)
```

and the corresponding Slick table class,

```scala
  class AuthorBooks(tag: Tag) extends Table[AuthorBook](tag, "author_book") {
    def authorId = column[Int]("author_id")
    def bookId = column[Int]("book_id")
    def authorFk = foreignKey("author_fk", authorId, authors)(_.id, onUpdate=ForeignKeyAction.Cascade, onDelete=ForeignKeyAction.Cascade)
    def bookFk = foreignKey("book_fk", bookId, books)(_.id, onUpdate=ForeignKeyAction.Cascade, onDelete=ForeignKeyAction.Cascade)
    def pk = primaryKey("pk", (authorId, bookId))
    def * = (authorId, bookId) <> ((AuthorBook.apply _).tupled, AuthorBook.unapply)
  }
  val authorBooks = TableQuery[AuthorBooks]
```

Now lets add some author books.
```scala
// make sure to add the table to our in memory DB
(authors.schema ++ books.schema ++ authorBooks.schema).create,

// Create normalized many to many relationships
authorBooks += AuthorBook(stephenKing.id, theShining.id),
authorBooks += AuthorBook(jkRowling.id, harryPotter.id),
authorBooks += AuthorBook(jrrTolkien.id, theLordOfTheRings.id),
authorBooks += AuthorBook(danBrown.id, theDaVinciCode.id),
authorBooks += AuthorBook(jkRowling.id, fictitiousBook1.id),
authorBooks += AuthorBook(jrrTolkien.id, fictitiousBook1.id),
authorBooks += AuthorBook(stephenKing.id, fictitiousBook2.id),
authorBooks += AuthorBook(danBrown.id, fictitiousBook2.id),
```

Now that all our data including the relational many to many data is in place we can query it,

```scala
val futureResult = setupFuture.flatMap { _ =>
  // Read all books by Stephen King
  println("Stephen Kings Books:")
  val booksJoinedToAuthorBooks = books join authorBooks on (_.id === _.bookId)
  val booksFilteredToStephenKing = booksJoinedToAuthorBooks.filter(_._2.authorId === stephenKing.id)
  db.run(booksFilteredToStephenKing.result).map(_.map(_._1).foreach(println))
}.flatMap { _ =>
  // Read all books by Stephen King
  println("Fictitious Book 2's Authors:")
  val authorsJoinedToAuthorBooks = authors join authorBooks on (_.id === _.authorId)
  val authorsFilteredToFictitiousBook2 = authorsJoinedToAuthorBooks.filter(_._2.bookId === fictitiousBook2.id)
  db.run(authorsFilteredToFictitiousBook2.result).map(_.map(_._1).foreach(println))
}
```

Here we are joining the associative table with the table we are interested in and applying a filter to it.

I personally think it is very cool the way slick allows you to create very declarative functional queries against
defined table schema classes with type safety (which I could not overstate my praises for).

Finally putting all the work together into a single runnable file,

```scala
package example

import slick.jdbc.H2Profile.api._
import scala.concurrent.Await
import scala.concurrent.duration.Duration

import scala.concurrent.ExecutionContext.Implicits.global

case class Author(id: Int, name: String)
case class Book(id: Int, name: String)
case class AuthorBook(authorId: Int, bookId: Int)

object Hello extends App {

  class Authors(tag: Tag) extends Table[Author](tag, "author") {
    def id = column[Int]("id", O.PrimaryKey)
    def name = column[String]("name")
    def * = (id, name) <> ((Author.apply _).tupled, Author.unapply)
  }
  val authors = TableQuery[Authors]

  class Books(tag: Tag) extends Table[Book](tag, "book") {
    def id = column[Int]("id", O.PrimaryKey)
    def name = column[String]("name")
    def * = (id, name) <> ((Book.apply _).tupled, Book.unapply)
  }
  val books = TableQuery[Books]

  class AuthorBooks(tag: Tag) extends Table[AuthorBook](tag, "author_book") {
    def authorId = column[Int]("author_id")
    def bookId = column[Int]("book_id")
    def authorFk = foreignKey("author_fk", authorId, authors)(_.id, onUpdate=ForeignKeyAction.Cascade, onDelete=ForeignKeyAction.Cascade)
    def bookFk = foreignKey("book_fk", bookId, books)(_.id, onUpdate=ForeignKeyAction.Cascade, onDelete=ForeignKeyAction.Cascade)
    def pk = primaryKey("pk", (authorId, bookId))
    def * = (authorId, bookId) <> ((AuthorBook.apply _).tupled, AuthorBook.unapply)
  }
  val authorBooks = TableQuery[AuthorBooks]

  // Create a connection to our in-memory database
  val db = Database.forConfig("h2mem1")

  val stephenKing = Author(1, "Stephen King")
  val jkRowling = Author(2, "J. K. Rowling")
  val jrrTolkien = Author(3, "J. R. R. Tolkien")
  val danBrown = Author(4, "Dan Brown")

  val theShining = Book(1, "The Shining")
  val harryPotter = Book(2, "Harry Potter")
  val theLordOfTheRings = Book(3, "Lord of the Rings")
  val theDaVinciCode = Book(4, "The Da Vinci Code")
  val fictitiousBook1 = Book(5, "Ficticious Book 1")
  val fictitiousBook2 = Book(6, "Ficticious Book 2")

  try {
    val setup = DBIO.seq(
      // Create the tables
      (authors.schema ++ books.schema ++ authorBooks.schema).create,

      // Insert some authors
      authors += stephenKing,
      authors += jkRowling,
      authors += jrrTolkien,
      authors += danBrown,

      // Insert some books
      books += theShining,
      books += harryPotter,
      books += theLordOfTheRings,
      books += theDaVinciCode,
      books += fictitiousBook1,
      books += fictitiousBook2,

      // Create normalized many to many relationships
      authorBooks += AuthorBook(stephenKing.id, theShining.id),
      authorBooks += AuthorBook(jkRowling.id, harryPotter.id),
      authorBooks += AuthorBook(jrrTolkien.id, theLordOfTheRings.id),
      authorBooks += AuthorBook(danBrown.id, theDaVinciCode.id),
      authorBooks += AuthorBook(jkRowling.id, fictitiousBook1.id),
      authorBooks += AuthorBook(jrrTolkien.id, fictitiousBook1.id),
      authorBooks += AuthorBook(stephenKing.id, fictitiousBook2.id),
      authorBooks += AuthorBook(danBrown.id, fictitiousBook2.id),
    )

    val setupFuture = db.run(setup)

    val futureResult = setupFuture.flatMap { _ =>
      // Read all authors and print them to the console
      println("Authors:")
      db.run(authors.result).map(_.foreach(println))
    }.flatMap { _ =>
      // Read all books and print them to the console
      println("Books:")
      db.run(books.result).map(_.foreach(println))
    }.flatMap { _ =>
      // Read all normalized relationships
      println("AuthorBooks:")
      db.run(authorBooks.result).map(_.foreach(println))
    }.flatMap { _ =>
      // Read all books by Stephen King
      println("Stephen Kings Books:")
      val booksJoinedToAuthorBooks = books join authorBooks on (_.id === _.bookId)
      val booksFilteredToStephenKing = booksJoinedToAuthorBooks.filter(_._2.authorId === stephenKing.id)
      db.run(booksFilteredToStephenKing.result).map(_.map(_._1).foreach(println))
    }.flatMap { _ =>
      // Read all books by Stephen King
      println("Fictitious Book 2's Authors:")
      val authorsJoinedToAuthorBooks = authors join authorBooks on (_.id === _.authorId)
      val authorsFilteredToFictitiousBook2 = authorsJoinedToAuthorBooks.filter(_._2.bookId === fictitiousBook2.id)
      db.run(authorsFilteredToFictitiousBook2.result).map(_.map(_._1).foreach(println))
    }

    Await.result(futureResult, Duration.Inf)
  } finally db.close

}
```

Even if you are not a Slick expert I hope this post gives you a taste of how Slick can be used with normalized data
models.

As always, feel free to ping me with any questions or feedback you might have for the post.

Thanks!
