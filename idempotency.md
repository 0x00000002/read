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


## The key to the potentialities

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



