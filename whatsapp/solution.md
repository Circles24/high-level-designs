# Whatsapp / Messenger 

## Requirements
Core Requirements
- User should be able to write messages to another user
- System should support groups ie collection of users
- User should be able to write to users
- Strong consistency with group presence ie if someone is removed from the grp, he should not be able to receive a message

## Non functional requirements
Core Requirements
- The system should support 1 Billion users
- Groups can have upto 100 members
- Message delivery latency should be as low as possible
- Message delivery throughput should be high
- The system should be fault tolerant
- The system should favour availability wrt to messages and delivery

## Entities

- User
- Chat
- Chat Membership
- Message
- Inbox

## APIs
- GET /user/{id} - get user details
- GET /user/chat - get all the chats this user is part of
- GET /chat/{id} - get chat by id
- Socket connection to write and read messages

## Observations and design highlights
1. We cannot employ periodic polling from client side for receiving messages
2. We should be establishing a bi-directional socket connection for sending and receiving messages
3. Cassandra makes sense for message table, partition key to be chat id, sort key to be creation timestamp
4. If a user is not connected to the system, we should send push notifications to the client application for the message 
5. For chat membership shareded postgres on chat id makes sense
6. We should keep a CDC based eventually consistet table for user to chats mapping : note this would be eventually consistent
7. We have a problem when fanning out messages to recipients, we dont know which socker server this user might be on so, we can utilise a Redis based pub sub solution
8. All the interested parties can subscribe into a topic in our case it would be user id, when fanning out we'll write the message to the user id topic of recipient
9. We'll have a cleanup job to clean up media content from the object store, and also to cleanup stale messages

## Datastore
1. Shareded RDBMS on user id for user table
2. Dynamodb for chat table ie just a key read pattern
2. Shareded RDBMS on chat id for chat membership table
3. Cassandra for message table

## Architecture diagram

![Architecture diagram](./assets/whatsapp.drawio.svg "Architecure diagram")

## Components
1. User service
2. Chat service
3. Message service
4. Socker hub service

## Flows

1. User sending message flow
    - User connects to the system
    - A socket connection is established between the user and the socket hub
    - Socket hub instance subscribes to the user id topic in the Redis pub sub
    - User writes a message
    - Socket hub forwards it to message service which write it to message table
    - CDC picks up the message, flink fetches the recipients associated for a message
    - Flink writes the message into inbox table for all recipients
    - Flink writes the message to the redis pub sub if user is online
    - If user is offline Flink forwards it to push notification APIs
    - All the interested Socket hub server picks up the message and forwards it to the recipient socket connections
2. User logging in after being offline
    - User connects to the system
    - A socket connection is etup between the client and the socket hub
    - System drains the inbox table to the client initially
    - Post the drain system starts forwarding the live messages


