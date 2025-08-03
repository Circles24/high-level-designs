# Ticket master

## Requirements
Core Requirements
- Users should be able to view events
- Users should be able to search for events
- Users should be able to book tickets to events
- Users should be able to view their booked events
- Admins or event coordinators should be able to add events
- Popular events should have dynamic pricing

## Non functional requirements
Core Requirements
- The system should prioritize availability for searching & viewing events, but should prioritize consistency for booking events (no double booking)
- The system should be scalable and able to handle high throughput in the form of popular events (10 million users, one event)
- The system should have low latency search (< 500ms)
- The system is read heavy, and thus needs to be able to support high read throughput (100:1)
- The system should be fault tolerant
- The system should provide secure transactions for purchases
- The system should have regular backups

## Entities

- User
- Performer
- Venue
- Event
- Seat
- Ticket

## APIs
- GET /event - list popular events
- GET /event/{id} - get event by id
- GET /user/ticket - get users tickets
- GET /user/ticket/{id} - get ticket by id
- GET /event/{id}/seat - get seats for given event
- GET /event/search/{query} - full text search on events
- POST /event - admin endpoint to create event

## Observations and design highlights
1. Search to be implemented using inverted index eg elastic search which would be build over the event data using change data capture
2. Event table write throughput would be less, reads would be very high, so it should be read optimised
3. We can utilise single leader multi follower RDMS cluster for events table
4. We can utilise redis cache for popular events
5. Since booking is a critical system and we need consistency there, we'll opt for RDBMS cluster, which would be sharded basis hash of event id
6. Tickets can again be stored in RDBMS cluster with index on user id and event id, so that reads are efficient and db can be sharded basis user id

## Datastore
1. CDN for media content
2. RDBMS shareded on user id for users table
3. RDBMS sharded on event id for bookings table
4. Redis cache for popular events

## Architecture diagram

![Architecture diagram](./assets/ticket-master.drawio.svg "Architecure diagram")

## Components
1. Event service
2. Search service
3. Booking service
4. Ticket servicec

## Flows

1. User ticket booking flow
    - User logs into the app
    - System fetches the home feed and renders
    - User searches the event
    - System employs full text search and finds the matching events list
    - User picks the correct event
    - System fetches the event details by using event id and renders the event screen
    - Media content is fetched from CDN
    - User clicks on book ticket
    - System fetches the current map of the venue and establishes a socket connection with the server
    - Socket connection keeps the client updated on the seatch occupancy changes, eg some seat might get locked, some might be unlocked
    - User picks up the seats and start the payment flow
    - User booking is captured and commited into the DB
    - CDC picks the user booking and generates the ticket basis the booking


