# Foreword
Hi 👋 I thought the question asked was more interesting than the usual LinkedIn messages, so I spent roughly 45 minutes after work to give it some thought, as I am not too familiar with FinTech it is a bit hard to do without anybody answering my own questions 😅.  
If you are interested in interviewing me after reading this feel free to reach out:  
LinkedIn: https://www.linkedin.com/in/meron-nagy-15350920b/  
Email: nagy.meron@gmail.com

___

# Question
We're currently e.g. working on the order life cycle: that means we're building an order management system and an exchange connection that is seamless and provides real-time access to live pricing and instrument data. I'm interested to hear how you would build this?

___

## My understanding of the initial question
The project would consist of 3 components or services:
1. Exchange connection - Responsible for communication with one or more exchanges.
2. Order management system (OMS) - Manages orders.
3. End-user facing application - Interface for users to interact with the system.

Each component/service needs to be low-latency, stable, reliable and secure.

To ensure reliability we would also need monitoring and logging systems.

___

## Questions I would ask in an Interview
### General
1. What does real-time mean in this context? What are the latency expectations?
2. What is the scale of the system?
    1. Is this for Germany, DACH, EU, Global?
    2. How many concurrent users are we planning for?
    3. How many orders per second do we expect?
    4. Are there performance benchmarks? E.g. order processing within X ms
3. Are we integrating into existing systems besides Exchanges?
4. Are there specific regulations that we have to follow?

### Exchange connection
1. Do we connect a single or multiple exchanges?
2. Are there rate limits or throttling on the exchange side?
3. What data formats do the exchanges support? FIX?

### OMS
1. What order types are we supporting? Market, Limit, Stop?
2. Should we support order modifications and cancellations when possible?
3. I imagine that this is highly regulated are there any compliance requirements at any step e.g. before sending an order?
4. Do we need an audit trail or similar for compliance?
5. Are we supporting real-time monitoring and alerting/notifications on order execution?
6. If we support multiple exchanges do we implement Smart Order Routing (SOR)?

___

## Exchange Connection
I know that exchanges commonly use FIX Protocol over TCP. While I have very limited experience in FinTech, I would assume the Exchange Connection needs to handle large volumes of real-time trading data.  
The Exchange Connection needs to communicate with both the OMS and the end-user application.
I would recommend event-driven architecture using Kafka as real-time data streaming system for these reasons:
- It is designed for high volume, low-latency messaging perfect for real-time updates.
- Provides strong durability and fault-tolerance
- Kafka is very popular and used by many companies, this means there is a large ecosystem for integration and tooling
- Using Kafka we can also replicate messages this would be useful for audit purposes by saving a copy of each message in long-term storage or allow replaying messages for reprocessing events if needed.

___

## OMS
The system should support creating, updating, and canceling orders in real time.  
The system needs persistent storage here PostgresQL would be an ideal choice, it is one of the most used general use databases, extremely reliable it will cover most if not all requirements from complex queries for analysis to database replication.  
For data that is frequently queried Redis could serve as a caching layer.  
Redis would help minimize database load and provide faster response times than PostgreSQL

___

## End-user facing application
Here I imagine a web application where we are again using event-driven architecture.  
We have a Frontend and Backend that communicate via WebSockets. The Backend communicates with Exchange Connection and OMS using Kafka.

### Real-Time Data Delivery with WebSockets
Because WebSockets are persistent and bidirectional it eliminates the need for repeated HTTP requests and the overhead of establishing connections that we would have with a traditional HTTP API reducing latency.

This means that as soon as the market data changes (e.g., a stock price updates), the backend can push the new data to the users browser instantly, without the user having to refresh the page or the client to manually poll for updates.

### Alternative: using QUIC (HTTP/3)
In the case where low latency would be the most important factor ignoring development time and cost QUIC would be the ideal choice in theory.  
Because QUIC uses UDP it should be faster as it gets rid of head-of-line blocking which further reduces latency.

There are 2 main reasons why I cannot recommend QUIC at this time:
1. QUIC is not widely supported yet and the ecosystem is still lacking as it is fairly new, this means it is inherently more complex to implement
2. There is no guarantee the actual QUIC implementation will be faster than a WebSockets implementations, there are cases in which HTTP/2 remains faster than HTTP/3.

### WebSocket Automatic Reconnection
The Client needs to be able to detect when a connection is lost and automatically attempt to reconnect.
Here, an exponential backoff retry strategy would work.

### Security / Authentication
We have to use the WebSocket Secure `wss://` version of the protocol because the regular WebSocket protocol `ws://` is transmitted in plaintext.  
For authentication, we can use JWT because they are stateless they are an excellent choice for distributes systems this makes them a better choice than session based authentication options.
JWTs also expire unlike API keys so in a potential scenario where a JWT would be stolen it would not be as severe as a stolen API key.

### Limitations
Depending on how much data a user can see at any given time we have to consider throttling and/or aggregating data.

___

## Tech Stack
### Frontend
- Svelte, Vite, WebSocket API

I would choose Svelte because it eliminates the overhead of a virtual DOM, which makes it faster in updating elements compared to React. In my experience it is also easier to work with.

Alternative: React is the industry standard, and it is a lot easier to find developers familiar with it.

### Backend
- Go with Gorilla WebnSocket

Go would be my preferred choice because of its high concurrency model and wide usage in real-time applications despite having a garbage collector.  
Gorilla WebSocket is well-tested, fast and widely adopted.

Alternative: Rust with FastWebSockets is one of the fastest options for WebSockets and does not have the garbage collector overhead that Go does. The downside here being that Rust is more complex and yet again it is harder to find developers familiar with it.

### Database
- PostgreSQL for Order Management & Market Data

If the project involves financial transactions or settlement:
- TigerBeetle is a database designed for financial transactions with builtin double-entry accounting, which would be useful for high-performance transaction processing. But this may not be necessary for this project.

### Cache
Redis as explained previously

### Message Bus
Kafka as explained previously.

### Monitoring
Prometheus/Grafana for real-time monitoring of each Service.