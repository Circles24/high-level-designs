# Uber 

## Requirements
Core Requirements
- User should be able to request rides for given source and destination
- Driver should be able to request rides
- System should match the ride request on the fly
- User should be given initial time and cost estimates

## Non functional requirements
Core Requirements
- The system should be highly available (prioritizing availability over consistency).
- The system should support high number of concurrent ride requests.
- The system should be able to prioritise the drivers basis the distance from the start location.

## Entities

- User
- Driver
- Ride request
- Ride availability request
- Ride

## APIs
- POST /driver/location - post drivers location
- POST /ride/request - submit a ride request
- GET /ride/request/{id} - poll ride request
- GET /ride/{id} - get ride details

## Observations and design highlights
1. Drivers will keep sharing thier locations using the driver app
2. System would maintain a geospatial index of all the available drivers
3. Once a user request a ride, system would figure out the applicable drivers and will ask them for ride confirmation using a push / in app notification
4. Once a driver confirms a ride request, he/she would be allocated to that ride and would be removed from the geospatial index
5. System would use a machine learning model to estimate cost basis multiple factors like time, location, traffic conditions, time estimate, etc
6. System would use an external map service  to figure out the time for navigation and for driver navigation.

## Datastore
1. Redis based geospatial index for drivers
2. Postgres RDS for users and drivers tables
3. Dynamodb for rides table

## Architecture diagram

![Architecture diagram](./assets/uber.drawio.svg "Architecure diagram")

## Components
1. Location service
2. Ride matching service
3. Ride service
4. Map and navigation service
5. User service
6. Driver service
7. Notification service

## Flows

1. Driver starts accepting requests
    - Driver logs in
    - Driver start accepting requests
    - Application setup a socket connection
    - App starts sharing live driver location
    - System keeps indexing the driver location
2. User requests ride
    - User logs in
    - User requests ride
    - System calls the maps service and figure out the distance and ETA for travel
    - System calls the cost estimate ML model to figure out the approx cost
    - System shares these details with the user
    - User procceds with the ride matching
    - System pushes a request for ride matching in the ride matching queue
    - Matching engine picks up the request
    - Engine picks up the drivers near the start locations and starts sending them ride requests one by one
    - Once a driver accepts the ride, system initiates a ride and notify the user

