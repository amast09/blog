+++
date = "2017-07-30T11:36:35-05:00"
title = "Advanced Scala For Comprehensions"
description = "Using Scala for comprehensions to untangle dependent asynchronous resources"
tags = [ "Scala", "for-comprehensions", "asynchronous" ]
categories = [ "Development", "Scala", "How To" ]
+++

Working with multiple dependent asynchronous Futures in Scala can be tough, especially when their results need to be transformed.
Thankfully Scala has for comprehensions to help unwind your code and make it more manageable.

In this post I am going to show through example how to work through an advanced dependent series of Futures.

Our fictitious problem will be calculating the average age of a movie director for a given movie rating.
In the real world this could probably be done using straight SQL however it provides us a quick and easy to understand situation.

Here are the case classes we will be working with,

```scala
case class Movie(name: String, year: Int, rating: BigDecimal, directorId: Int)

case class Director(id: Int, name: String, yearBorn: Int)
```

Here are our fake asynchronous interfaces we will work with,

```scala
def getMoviesByRating(rating: BigDecimal): Future[Seq[Movie]] = {
  val fakeDbOfMovies = Seq(
    Movie("The Shawshank Redemption", 1994, 9.2, 1),
    Movie("The Godfather", 1972, 9.2, 2),
    Movie("The Godfather: Part II", 1974, 9.0, 2),
    Movie("The Dark Knight", 2008, 9.0, 3),
    Movie("12 Angry Men", 1957, 8.9, 4),
    Movie("Schindler's List", 1993, 8.9, 5),
    Movie("Pulp Fiction", 1994, 8.9, 6),
    Movie("The Lord of the Rings: The Return of the King", 2003, 8.9, 7),
    Movie("The Good, the Bad, and the Ugly", 1966, 8.9, 8),
    Movie("Fight Club", 1999, 8.8, 9)
  )
  Future.successful(fakeDbOfMovies.filter(_.rating == rating))
}

def getDirector(directorId: Int): Future[Option[Director]] = {
  val fakeDbOfDirectors = Seq(
    Director(1, "Frank Darabont", 1959),
    Director(2, "Francis Ford Coppola", 1939),
    Director(3, "Christopher Nolan", 1970),
    Director(4, "Sidney Lumet", 1924),
    Director(5, "Steven Spielberg", 1946),
    Director(6, "Quentin Tarantino", 1963),
    Director(7, "Peter Jackson", 1961),
    Director(8, "Sergio Leone", 1929),
    Director(9, "David Fincher", 1962),
  )
  Future.successful(fakeDbOfDirectors.find(_.id == directorId))
}
```

Given these parameters / situation a naive solution not using for comprehensions may look like this,

```scala
def getAverageAgeOfDirectorForRatingNested(rating: BigDecimal): Future[Int] = {
  val currentYear = Calendar.getInstance().get(Calendar.YEAR)

  getMoviesByRating(rating).flatMap(moviesWithRating => {
    val directorAgesForRating = moviesWithRating.map(director => {
      getDirector(director.directorId).map(maybeDirector => {
        maybeDirector.map(director => {
          currentYear - director.yearBorn
        })
      })
    })

    Future.sequence(directorAgesForRating).map(_.flatten[Int].sum / directorAgesForRating.length)
  })
}
```

The solution is highly nested and it is very hard to pick apart what is going on.

Scala sequence comprehensions are very powerful however if you are not careful you can quickly make an unintelligible mess with them as you can see above.

Let's jump right into a cleaner, easier to comprehend and more maintainable solution using a for comprehension.

```scala
def getAverageAgeOfDirectorForRating(rating: BigDecimal): Future[Int] = {
  val currentYear = Calendar.getInstance().get(Calendar.YEAR)

  for {
    moviesWithRating <- getMoviesByRating(rating)
    directorIdsForRating <- Future.successful(moviesWithRating.map(_.directorId))
    maybeDirectors <- Future.sequence(directorIdsForRating.map(getDirector))
    directors <- Future.successful(maybeDirectors.flatten[Director])
    directorBirthYears <- Future.successful(directors.map(_.yearBorn))
    directorAges <- Future.successful(directorBirthYears.map(currentYear - _))
    averageDirectorAge <- Future.successful(directorAges.sum / directors.length)
  } yield averageDirectorAge
}
```

In this solution we are still using Scala's powerful sequence comprehensions but we wrap them in a for comprehension and are sure to give each step in the sequence a meaningful name.

The trick to mixing Future's with general sequence transformations and calculations is the fact that you need to wrap them inside of a Future themselves so that the comprehension can use their results.

The other thing to note is the `Future.sequence` call. This is transforming the sequence of Future Directors into a single Future containing a sequence of Directors which I think is very cool!

How I approach problems like this is to first get a working solution. Ideally I get a working solution by writing a failing test, then getting that test to pass.
After that baseline is achieved I iterate on the solution, cleaning it up, making it simpler and easier to understand.
Since I have a test in place I can quickly understand if my iterated solution is still correct.

Here is a Github gist containing all the code from this post.

https://gist.github.com/amast09/52a412ec814b613e072d20bfe8c5f487

You can easily pop it into a Scala worksheet in Intellij and experiment with the code.

As always, please let me know if you have any feedback, suggestions or improvements to this post or the code.
Feel free to leave comments on the Github gist.

Cheers!
