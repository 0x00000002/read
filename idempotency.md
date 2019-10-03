# Intern Vasya and his stories about the API’s idempotency

  *Idempotency* - sounds difficult, they say it rarely, but it applies to all applications that use the API in their work.


My name is Denis Isaev and I run one of the backend groups in Yandex. Today I will share with Habr’s readers a description of the problems that can occur if you don’t take into account the distributed systems’ idem-potency in your project. To do this, I chose the format of fictional stories about trainee Vasya, who is just learning to work with API. It will be clearer and more useful in this way. Let’s go.

## About API

Vasya was developing an application for ordering a taxi from scratch and got the task of making an API for ordering a car. He sat around the clock and implemented an API like POST /v1/orders:

```
{
  "from": "Moscow, 82c2 Sadovnicheskaya Embankment",
  "to": "Vnukovo Airport"
}
```

When it was necessary to make an API for active orders, Vasya thought: “Would it be necessary to order several taxi cars at once? The managers answered that they did not need such an opportunity. However, he made an API to give out the list of active orders in general GET /v1/orders:

```
{
  "orders." [
    {
      "id": 1,
      "from": "Moscow, 82c2 Sadovnicheskaya Embankment",
      "to": "Vnukovo Airport"
    }
  ]
}
```

In the mobile application, Fedya’s programmer supported the server API as follows:


when starting the application, call GET /v1/orders, and if you receive an active order, draw it in the UI state;
when clicking on the “order a taxi” button, call POST /v1/orders with the entered user data;
if any server or network error occurs, draw an error message and do nothing else.

As it should be, the server code and application code were autotested, and before the release of the mobile application it was manually tested for 2 days. The testing found a number of bugs and they were quickly fixed. The application was successfully stabilized on users and gave an advertising campaign. Users left some positive feedback, thanked the developers and asked for new features. Development team and managers noted the successful launch of donuts and went home.


## Blocking the button


At 8 a.m., Vasya was woken up by a call from the sapport: two users complained that two cars arrived instead of one, and the money was written off for both cars. Quickly making coffee, Vasya sat down at the laptop, connected via VPN and started digging logs, graphics and code. Vasya found out from the logs that these users had two identical requests with a difference of a few seconds. According to the charts, he saw that at 7 a.m., the database began to slow down and the requests for records in the database began to work in seconds instead of milliseconds. By this point, the cause of the slow requests has already been found and eliminated, but there is no guarantee that this will not happen again someday. And then he realized: the application does not block the button “order a taxi” after sending the request, and when the requests began to slow down, users began to press the button again, thinking that the first time it was not pressed.


![button](https://habrastorage.org/webt/ic/ch/fk/icchfk11fe8siv3sdbp8emzyyqo.png)


The application began to block the button: this fix was stained after a few days. But the team had to receive similar complaints for a few more weeks and ask users to update the application.


## In the underground tunnel

Another similar complaint came in, and the inertia sapport-reply “update the app”. But here the user said that he already has the newest version of the application. Wasyu and Fedya were pulled out of their current bugs and asked to find out how it was done, because this bug had already been fixed.

Having spent two days digging up this single case, they found out what was going on. It turned out that blocking the button wasn’t enough: one of the users tried to order a taxi while in an underground passage. His mobile Internet was barely working: when he pressed the order button, the request went to the server, but the answer was not received. The application showed the message “an error occurred” and unblocked the order button. Who would have thought that such a request could have been successfully executed on the server, and the taxi driver was already on his way?

We chose the option to edit on the server, as it can be done on the same day, without waiting for a long roll of the application. From some variants of correction Vasya has chosen such: before creation of the order in base it will select from base of orders of the user with the same parametres from and to for last 5 minutes. If such an order is found, the server gives an error of 500. Vasya wrote autotests and, by chance, ran them in parallel: one of the tests fell down. Vasya realized that there is a race between the selector and the insert in the database when there are parallel requests from one user. According to the results of the bugs Vasya realized that the network can “blink” and the database can slow down, increasing the window of the race, so the case is quite real. How to fix it correctly was not clear.


## Limits on the number of active orders

On the advice of a more experienced programmer, Vasya looked at the problem from the other side and bypassed the race using this algorithm:

- start transaction;
- `UPDATE active_orders SET n=1 WHERE user_id={user_id} AND n=0;`
- if update has changed 0 records, then return the HTTP code 409;
- insert the order object into another table;
- complete the transaction

The application re-requested the list of active orders when receiving the 409 response code. The fix on the server was stabilized on the same day, duplicated, and after downloading the application users stopped seeing errors. Vasya and Fedya returned to their features.


## Multi-order

A month passed and a new manager came to Vasya: how many days do you have to make a "multi-order" feature: so that the user can order two taxi cars? Vasya is surprised: how so, I asked, and you told me that it would not be necessary?! Vasya said that this is not fast. The manager was surprised: isn't it just to raise the limit from 1 to 2? But the multi-order completely broke Vasina's scheme of protection against takeovers. Vasya did not even know how to solve this problem at all, without introducing takes.


![multiorder](https://habrastorage.org/webt/j8/r2/_x/j8r2_xp789ymsnaslh8z5e81gk8.png)


## The idempotency key 

Vasya decided to study how to deal with such problems, and came across the notion of idematency. Idempotent is a method called API, the repeated call of which does not change the state. There is a subtle point here: the result of a potential call may change. For example, if you call the call again to create an order, the order will not be created again, but the API can answer both 200 and 400. With both API response codes, the API will be empathetic in terms of the state of the server (order one, nothing happens to it), and from the client's point of view, the behavior is significantly different.

Also Vasya found out that HTTP methods GET, PUT, DELETE are formally considered to be potential-free, while POST and PATCH are not. This does not mean that you cannot make GET unidempotent, but POST idempotent. But this is something that many programs rely on, for example, proxy servers may not repeat POST and PATCH requests in case of errors, while GET and PUT may repeat.

Vasya decided to look at examples and came across the concept of idempotency key in some public APIs.

Yandex allows clients to send the Idempotency-Key header with a unique key generated by the API client together with formally non-IDempotency (POST) requests. It is recommended to use UUID V4. Stripe similarly allows clients to send the Idempotency-Key header with the unique key generated by the API client together with formally non-IDempotent (POST) requests. The keys are stored for 24h. Among the non-payment systems Vasya found the client tokens at AWS.


Vasya added a new mandatory field idempotency_key to the POST /v1/orders request, and the request became so:

```
{
  "from": "Moscow, 82c2 Sadovnicheskaya Embankment",
  "to": "Vnukovo Airport."
  "idempotency_key": "786706b8-ed80-443a-80f6-ea1fa8cc1b51
}
```

The application began to generate a key of capacity as UUID v4 and send it to the server. At repeated attempts to create an order, the application sends the same key and capacity. On the server the key of the capacity key is stored in the database in the field where there is a limitation of the database by its uniqueness. If this restriction prevented the insertion, the code detected it and gave an error of 409. On Fedi's advice, this moment was redesigned to simplify the application: it was not 409, but 200, as if the order was successfully created, then the clients do not need to learn how to process the code 409.


![image](https://habrastorage.org/webt/ow/tx/gu/owtxguxu8hwtyymgkkijm6-4s8k.png)


## Bug in testing

The limit was then simply raised from 1 to 2 and supported the change in the annex. The following bug was found while testing the application:


1. The user wants to create an order, the request comes to the server, the order is created, the tester emulates a network error and the application does not receive a response;
2. The user sees the error message and for some reason changes the destination beforehand, and only after pressing the taxi button again;
3. The application does not change the key and capacity between requests;
4. The server detects that the order with this idempotency key is already there and gives away 200;
5. the server has created an order with an old destination, and the user thinks he is created with a new destination, and leaves the wrong place.

First, Vasya proposed to the Federation to generate a new key and capacity in this case. But Fedya explained that then there can be a take: in case of a network error of the order creation request, the application cannot know whether the order was actually created.


Fedya noticed that although this is not a solution, to detect such bugs early on the server it was necessary to check that the parameters of the incoming request coincide with the parameters of the existing order with the same key and capacity. For example, AWS gives an IdempotentParameterMismatch error in this case.


As a result, the two of them came up with the following solution: the application does not allow changing the order parameters and tries to create an order indefinitely while it receives 5xx response codes or network errors. Vasya added the server validation suggested by Fedya.


## Useful code review

Two problem scenarios were found on the code tree of the implemented solution.


### Scenario 1: two taxis

1. The application sends a request to create an order, the request is executed for tens of seconds for some reason, the order is slowly created;
2. The user can't do anything in the application without ordering a taxi, so he decides to unload the application from memory completely;
3. the user reopens the application, makes a request to GET /v1/orders, and does not receive the currently created order because it has not been created yet;
4. The user thinks that the application has glitched and makes the order again, this time the order is created quickly;
5. The creation of the first order sagged, and the order was created to the end;
6. Two taxis arrive at the passenger's place.

### Scenario 2: A cancelled taxi arrives

1. The application sends a request to create an order, the order is created, but the mobile network is attached, and the application does not receive an answer about the successful creation of the order;
2. The dispatcher or the user cancels the order via the fur for some reason: the order is canceled as deleting a line from the database table;
3. The application sends a second order creation request: the request is successfully executed and another order is created, because the key and potentialities stored in the previous order no longer exist in the table.

Vasya and Fedya considered simple options to fix both problems:

**Scenario 1:** The application stores all the orders that are currently being created, even between application restarts. The application shows them in the interface immediately after the start, continuing to try to create them, provided it hasn't been too long since they were created.

**Scenario 2:** move from deleting entries in the order table to setting the field Deleted_at=now(), the so-called soft delete. Then the limitation of the key uniqueness and capacity limitation would also work for cancelled orders.

**Scenario 3:** Separate the abstraction of the provisioning of the requests' capacity and the abstraction of the resources and store the used capacity and keys for a limited time separately from the resource, for example, 24h.

But the seniors suggested a more general solution: to version the state of the order list. The GET /v1/orders API would give a version of the order list. This is the version of the entire list of user orders, not the specific order. When creating an order, the application sends an If-Match version it knows about in a separate field or header. The server increases the version atomicly with the change in any changes of orders (creation, cancellation, editing). That is, the application tells the server what order status it knows in the request to the server. And if this state of orders (version) differs from what is stored on the server, the server gives an error "orders have been changed in parallel, reload the order information". Versioning solves both found problems, and it is his Vasya and Fedya and supported. It should also be noted that the version can be both a number (the number of the last change) and a hash from the list of orders: for example, the parameter fingerprint in Google Cloud API works to change the tags of instances.


## Time to draw conclusions

As a result of all the modifications, Vasya thought and understood that any resource creation API must be capacitated. In addition, it is important to synchronize knowledge of the list of resources on the client and the server through the versioning of this list.


## Deletion idempotency

One day Vasya receives a notification in telegrams that the API had a 404 response code. According to Vasya's logs, this happened in the order cancellation API.


Cancellation of the order was done through the request DELETE /v1/orders/:id. Inside the line with the order was simply removed. There was no need in soft delete (setting Deleted_at=now()).


In this situation, the application sent the first request to cancel, but it was staymowled. The application, without notifying the user, immediately made a request and received 404: the first request has already been executed and deleted the order. The user saw the message "unknown server error".


It turns out that not only the API of creation, but also the removal of resources should be capacitated, - thought Vasya.


Vasya considered the option to give 200 always, even if DELETE request in the database did not delete anything. But this created a risk to hide and miss possible problems. That's why he decided to make a soft delete and redo the cancellation API:


1. from the database, he began to select all the orders, even those already cancelled with this id;
2. If the order had already been deleted, and it was within the last n minutes (i.e. on the usual requests), the server began to give 200;
3. in other cases the server gives 410 with the error "order does not exist". Vasya decided to replace 404 with 410 as a more appropriate one, because the 404 code means that it is a temporary error, and the request can be repeated later. The 410 code means that the error is constant and the same result will be obtained by repeating the request.

There were no more such problems with canceling the order.


## Idempotency of the change

### Changing point B

Vasya decided to check by the code whether the API of the trip's change of pace was capacitated: he had already realized that absolutely any API should be capacitated.


![image](https://habrastorage.org/webt/ht/ck/dz/htckdzeycyvb3naiipkajenpzn8.png)



