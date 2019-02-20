#### Part23 : Application Programming Interfaces(APIs)

* `APIs` (Application Programming Interfaces)
  * a collection of HTTP routes that are designed as low-level entry points into the application
  * allow the client to work directly with the application's resources
  * *how to build `APIs` that do not rely on the web browser and make no assumptions about what kind of client connects to them??*



* REST as a Foundation of API Design

  * `REST API`

    * `REST` (Representational State Transfer) : six defining characteristics of REST

    ---

    ### Client-Server

    The client-server principle is fairly straightforward, as it simply states that in a REST API the roles of the client and the server should be clearly differentiated. In practice, this means that the client and the server are in separate processes that communicate over a transport, which in the majority of the cases is the HTTP protocol over a TCP network.

    ### Layered System

    The layered system principle says that when a client needs to communicate with a server, it may end up connected to an intermediary and not the actual server. The idea is that for the client, there should be absolutely no difference in the way it sends requests if not connected directly to the server, in fact, it may not even know if it is connected to the target server or not. Likewise, this principle states that a server may receive client requests from an intermediary and not the client directly, so it must never assume that the other side of the connection is the client.

    This is an important feature of REST, because being able to add intermediary nodes allows application architects to design large and complex networks that are able to satisfy a large volume of requests through the use of load balancers, caches, proxy servers, etc.

    ### Cache

    This principle extends the layered system, by indicating explicitly that it is allowed for the server or an intermediary to cache responses to requests that are received often to improve system performance. There is an implementation of a cache that you are likely familiar with: the one in all web browsers. The web browser cache layer is often used to avoid having to request the same files, such as images, over and over again.

    For the purposes of an API, the target server needs to indicate through the use of *cache controls*if a response can be cached by intermediaries as it travels back to the client. Note that because for security reasons APIs deployed to production must use encryption, caching is usually not done in an intermediate node unless this node *terminates* the SSL connection, or performs decryption and re-encryption.

    ### Code On Demand

    This is an optional requirement that states that the server can provide executable code in responses to the client. Because this principle requires an agreement between the server and the client on what kind of executable code the client is able to run, this is rarely used in APIs. You would think that the server could return JavaScript code for web browser clients to execute, but REST is not specifically targeted to web browser clients. Executing JavaScript, for example, could introduce a complication if the client is an iOS or Android device.

    ### Stateless

    The stateless principle is one of the two at the center of most debates between REST purists and pragmatists. It states that a REST API should not save any client state to be recalled every time a given client sends a request. What this means is that none of the mechanisms that are common in web development to "remember" users as they navigate through the pages of the application can be used. In a stateless API, every request needs to include the information that the server needs to identify and authenticate the client and to carry out the request. It also means that the server cannot store any data related to the client connection in a database or other form of storage.

    If you are wondering why REST requires stateless servers, the main reason is that stateless servers are very easy to scale, all you need to do is run multiple instances of the server behind a load balancer. If the server stores client state things get more complicated, as you have to figure out how multiple servers can access and update that state, or alternatively ensure that a given client is always handled by the same server, something commonly referred to as *sticky sessions*.

    **If you consider again the */translate* route discussed in the chapter's introduction, you'll realize that it cannot be considered *RESTful*, because the view function associated with that route relies on the `@login_required` decorator from Flask-Login, which in turn stores the logged in state of the user in the Flask user session.**

    ### Uniform Interface

    The final, most important, most debated and most vaguely documented REST principle is the uniform interface. Dr. Fielding enumerates four distinguishing aspects of the REST uniform interface: unique resource identifiers, resource representations, self-descriptive messages, and hypermedia.

    Unique resource identifiers are achieved by assigning a unique URL to each resource. For example, the URL associated with a given user can be */api/users/<user-id>*, where *<user-id>* is the identifier assigned to the user in the database table's primary key. This is reasonably well implemented by most APIs.

    The use of resource representations means that when the server and the client exchange information about a resource, they must use an agreed upon format. For most modern APIs, the JSON format is used to build resource representations. An API can choose to support multiple resource representation formats, and in that case the *content negotiation* options in the HTTP protocol are the mechanism by which the client and the server can agree on a format that both like.

    Self-descriptive messages means that requests and responses exchanged between the clients and the server must include all the information that the other party needs. As a typical example, the HTTP request method is used to indicate what operation the client wants the server to execute. A `GET` request indicates that the client wants to retrieve information about a resource, a `POST` request indicates the client wants to create a new resource, `PUT` or `PATCH` requests define modifications to existing resources, and `DELETE` indicates a request to remove a resource. The target resource is indicated as the request URL, with additional information provided in HTTP headers, the query string portion of the URL or the request body.

    The hypermedia requirement is the most polemic of the set, and one that few APIs implement, and those APIs that do implement it rarely do so in a way that satisfies REST purists. Since the resources in an application are all inter-related, this requirement asks that those relationships are included in the resource representations, so that clients can discover new resources by traversing relationships, pretty much in the same way you discover new pages in a web application by clicking on links that take you from one page to the next. The idea is that a client could enter an API without any previous knowledge about the resources in it, and learn about them simply by following hypermedia links. One of the aspects that complicate the implementation of this requirement is that unlike HTML and XML, the JSON format that is commonly used for resource representations in APIs does not define a standard way to include links, so you are forced to use a custom structure, or one of the proposed JSON extensions that try to address this gap, such as [JSON-API](http://jsonapi.org/), [HAL](http://stateless.co/hal_specification.html), [JSON-LD](https://json-ld.org/) or similar.

    ----



*  [HTTPie](https://httpie.org/), a command-line HTTP client written in Python that makes it easy to send API requests
* To simplify the interactions between client and server when token authentication is used, I'm going to use a Flask extension called [Flask-HTTPAuth](https://flask-httpauth.readthedocs.io/). Flask-HTTPAuth is installed with pip



---

Werkzung에 대해

https://juliahwang.kr/network/2017/09/17/WSGI%EB%9E%80%EB%AC%B4%EC%97%87%EC%9D%B8%EA%B0%80.html

