+++
date = "2017-09-30T17:34:18-04:00"
title = "Using async.js to Juggle Asynchronous Actions"
description = "Using the async.js JavaScript library to flatten asynchronous callback hell"
tags = [ "Javascript", "async.js", "asynchronous" ]
categories = [ "Development", "Javascript", "How To" ]
+++

JavaScript that has dependent asynchronous calls has a tendency to turn into
callback hell spaghetti code.

Now days a lot of this can be avoided by using promises instead of callbacks.
However if you want finer grained control over sequences of asynchronous operations
then async.js is a great library to use.

We will take a look at 3 very useful async method's that I have utilized on numerous
occasions. I will provide a solution to an example problem that does not utilize
async.js followed by a solution to the problem utilizing async.js.

Async.js operates with node style asynchronous callbacks so the callback is called
where the first parameter is an error object associated with the async operation
and a second parameter corresponding to the result of the async operation.

The first method we will take a look at is the `series` method,

https://caolan.github.io/async/docs.html#series

Here is a snippet of the relevant documentation,

> Run the functions in the tasks collection in series, each one running once the
> previous function has completed. If any functions in the series pass an error to
> its callback, no more functions are run, and callback is immediately called with
> the value of the error. Otherwise, callback receives an array of results when
> tasks have completed.

You can use this function when you have a number of non dependent async functions
that need to be run in a specific order. Our example scenario is a little bit
contrived but should hopefully give you an idea of what the method is and how
to use it. We will be emulating eating a meal at a restaurant and giving it a review.

Here are the async functions we will need to utilize,

```javascript
function eatFreeBread (bread, callback) {
  setTimeout(function () {
    console.log('Just ate some' + bread + ' bread');
    callback(null, {breadRating: 8});
  }, 3000);
}

function eatAppetizers (apps, callback) {
  setTimeout(function () {
    console.log('Just ate these great appetizers ' + apps);
    callback(null, {appRating: 9});
  }, 3000);
}

function eatMainCourse (mainCourse, callback) {
  setTimeout(function () {
    console.log('Just ate the ' + mainCourse + ' main course');
    callback(null, {mainCourseRating: 9});
  }, 3000);
}

function eatDessert (dessert, callback) {
  setTimeout(function () {
    console.log('Just ate the ' + dessert + ' dessert');
    callback(null, {dessertRating: 10});
  }, 3000);
}

function handlePoorService () {
  console.log('WE ARE DONE WITH THIS NON SENSE AND ARE LEAVING!')
}
```

We never want to start the next part of the meal until we are done with the
previous one. Here is a naive example solution of how we would solve it without
async.js.

```javascript
function goOutToEatV1 (bread, apps, mainCourse, dessert, restaurantReviewCallback) {

  eatFreeBread(bread, function (err, breadResult) {
    if (err) {
      handlePoorService();
      restaurantReviewCallback(err);
    } else {
      eatAppetizers(apps, function (err, appsResult) {
        if (err) {
          handlePoorService();
          restaurantReviewCallback(err);
        } else {
          eatMainCourse(mainCourse, function (err, mainCourseResult) {
            if (err) {
              handlePoorService();
              restaurantReviewCallback(err);
            } else {
              eatDessert(dessert, function (err, dessertResult) {
                if (err) {
                  handlePoorService();
                  restaurantReviewCallback(err);
                } else {
                  restaurantReviewCallback(null, [
                    breadResult,
                    appsResult,
                    mainCourseResult,
                    dessertResult
                  ]);
                }
              });
            }
          });
        }
      });
    }
  });
}
```

If that does not make your eyes bleed I do not know what will. Here is the same
code using async.js's series.

```javascript
function goOutToEatV2 (bread, apps, mainCourse, dessert, restaurantReviewCallback) {
  async.series([
    function (callback) {
        eatFreeBread(bread, callback);
    },
    function (callback) {
        eatAppetizers(apps, callback);
    },
    function (callback) {
        eatMainCourse(mainCourse, callback);
    },
    function (callback) {
        eatDessert(dessert, callback);
    },
  ], function (err, restaurantResults) {
    if (err) {
      handlePoorService();
    }
    restaurantReviewCallback(err, restaurantResults);
  })
}
```

We can do even better by using lodash's curry method to generate functions
that have there initial parameters supplied.

```javascript
function goOutToEatV3 (bread, apps, mainCourse, dessert, restaurantReviewCallback) {
  async.series([
    _.curry(eatFreeBread)(bread),
    _.curry(eatAppetizers)(apps),
    _.curry(eatMainCourse)(mainCourse),
    _.curry(eatDessert)(dessert),
  ], function (err, restaurantResults) {
    if (err) {
      handlePoorService();
    }
    restaurantReviewCallback(err, restaurantResults);
  })
}
```

The next method we will take a look at is the `map` method,

https://caolan.github.io/async/docs.html#each

Here is a quick snippet of their documentation for the method.

>Produces a new collection of values by mapping each value in coll through the
iteratee function. The iteratee is called with an item from coll and a callback
for when it has finished processing. Each of these callback takes 2 arguments:
an error, and the transformed item from coll. If iteratee passes an error to its
callback, the main callback (for the map function) is immediately called with
the error.

Our example problem for this method is much less contrived. We will send emails
to a list of email addresses in parallel and if any emails fail to send it will
stop trying to send them.

First the interfaces we will utilize.

```javascript
function sendEmail (emailAddress, email, callback) {
  setTimeout(function () {
    console.log('Sent the following email ' + email + ' to the following email address ' + emailAddress);
    callback(null, {emailId: Math.floor(Math.random() * 10000)});
  }, 3000);
}
```

Here is an example solution not using async.js,

```javascript
function sendBatchOfEmailsV1 (emailAddresses, emailBody, callback) {
  var emailHasFailed = false,
      emailAddressIdx = 0,
      emailIds = [];

  while (!emailHasFailed && emailAddressIdx < emailAddresses.length) {

    sendEmail(emailBody, emailAddresses[emailAddressIdx], function (err, result) {
      if (err) {
        emailHasFailed = true;
        return callback(err);
      } else {
        emailIds.push(result.emailId);
      }

      if (emailIds.length === emailAddresses.length) {
        callback(null, emailIds);
      }
    });

    emailAddressIdx += 1;
  }
}
```

This code isn't nearly as nested as the last example but there is still a lot
of nastiness to it as well as a lot of potential for logical errors.

Here is how we can implement the solution using async.js,

```javascript
function sendBatchOfEmailsV2 (emailAddresses, emailBody, callback) {
  async.map(emailAddresses, function (emailAddress, asyncCallback) {
    sendEmail(emailBody, emailAddress, asyncCallback);
  }, callback);
}
```

We can do even better if we utilize lodash's curry method.

```javascript
function sendBatchOfEmailsV3 (emailAddresses, emailBody, batchCallback) {
  async.each(emailAddresses, _.curry(sendEmail)(email), batchCallback);
}
```

Turning 25 lines of mess into a single line is a win in my book!

The last async method we will take a look at is the `waterfall` method,

https://caolan.github.io/async/docs.html#waterfall

Here is a quick snippet of their documentation for the method.

>Runs the tasks array of functions in series, each passing their results to the
next in the array. However, if any of the tasks pass an error to their own
callback, the next function is not executed, and the main callback is
immediately called with the error.

I probably use this method the most out of all of async's library. You can use
it for dependent async actions.

Our example problem will be booking an airline ticket.

Here are the interfaces we will utilize.

```javascript
function makeReservation (reservationDetails, callback) {
  setTimeout(function () {
    console.log('Made the following reservation ' + JSON.stringify(reservationDetails));
    callback(null, Math.floor(Math.random() * 10000));
  }, 3000);
}

function processCreditCardForReservation (creditCardDetails, reservationId, callback) {
  setTimeout(function () {
    console.log('Charged the following reservation ' + reservationId + ' to the following card ' + JSON.stringify(creditCardDetails));
    callback(null, Math.floor(Math.random() * 10000));
  }, 3000);
}

function emailAirlineTicketReceipt (emailAddress, receiptId, callback) {
  setTimeout(function () {
    console.log('Emailed a receipt for receipt ID ' + receiptId + ' to the following email address ' + emailAddress);
    callback(null, Math.floor(Math.random() * 10000));
  }, 3000);
}
```

First our implementation not utilizing async js,

```javascript
function bookAirlineTicketV1 (reservationDetails, creditCardDetails, receiptEmailAddress, bookAirlineTicketCallback) {
  makeReservation(reservationDetails, function (reservationErr, reservationId) {
    if (reservationErr) {
      bookAirlineTicketCallback(reservationErr);
    } else {
      processCreditCardForReservation(creditCardDetails, reservationId, function (creditCardProcessingError, receiptId) {
        if (creditCardProcessingError) {
          bookAirlineTicketCallback(creditCardProcessingError);
        } else {
          emailAirlineTicketReceipt(receiptEmailAddress, receiptId, function (emailError, emailId) {
              bookAirlineTicketCallback(emailError);
          });  
        }
      });
    }
  });
}
```

There is a lot of repeated code and nesting with this solution, we can improve
it a lot with async js.

```javascript
function bookAirlineTicketV2 (reservationDetails, creditCardDetails, receiptEmailAddress, bookAirlineTicketCallback) {
  async.waterfall([
      function(callback) {
          makeReservation(reservationDetails, callback);
      },
      function(reservationId, callback) {
          processCreditCardForReservation(creditCardDetails, reservationId, callback);
      },
      function(receiptId, callback) {
          emailAirlineTicketReceipt(receiptEmailAddress, receiptId, callback);
      }
  ], bookAirlineTicketCallback);
}
```

This has flattened our nested dependency structure but we can do even better
by utilizing lodash's curry function.

```javascript
function bookAirlineTicketV3 (reservationDetails, creditCardDetails, receiptEmailAddress, bookAirlineTicketCallback) {
  async.waterfall([
    _.curry(makeReservation)(reservationDetails),
    _.curry(processCreditCardForReservation)(creditCardDetails),
    _.curry(emailAirlineTicketReceipt)(receiptEmailAddress),
  ], bookAirlineTicketCallback);
}
```

Now that is much more concise than our original code.

You can mess with all of these code snippets using the Github Gist I created here,

https://gist.github.com/amast09/78d52d64ce54a171e118a4b94937ac88

I hope these examples are helpful and give a good understanding of how to use
a couple of async's methods as well as how powerful the library can be.

As always, please let me know if you have any feedback, suggestions or
improvements to this post or the code. Feel free to leave comments on the Github
gist.

Cheers!
