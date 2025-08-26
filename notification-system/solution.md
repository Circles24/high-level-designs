# Notifications System 

## Requirements
Core Requirements
- Services across the org should be able to send out notifications
- System to support Push, Email, SMS, Whatsapp notifications
- Services should also be able to schedule notifications at certain time
- Notifications should be near real time, ie system to fanout as soon as possible with slight delay acceptable when under high load
- Users should be able to opt out of the notifications
- System should ratelimit promotional messages based on config defined
- System to support notification templating
- Failed notifications should be retried with different their party vendor
- On fall back message failure, notifications should be parked for review and analysis

## Non functional requirements
Core Requirements
- The system should be highly available (prioritizing availability over consistency).
- The system should support very high number of concurrent notification requests.

## Entities

- Event Notification Config
- Event
- Notification
- SMS Notification
- Email Notification
- Push Notification
- Whatsapp Notification

## APIs
- POST /notification/event - submit an event for processing
- GET /notification/event/{eventId} - fetch notification status for notification
- GET /notification/sms/{id} - fetch sms notification status
- GET /notification/config/{eventType} - fetch notification config for an event type
- POST /notification/config/{eventType} - create / update notification config for an event type

## Observations and design highlights
1. This would be a classic fanout system.
2. The main capture API would capture a notification event and post it in an internal queue
3. The first layer of notification processing would be a dedup check layer
4. At the next step the notification orchestrator would fetch the event notification config
5. Basis config system would decide the channels to target
6. For each channel to target we'll create an event and push them to their respective queue
7. On message delivery failure either at our side or by getting error webhook status call, we'll reschedule message retry by sending the event to failure queue
8. System would handle failure reties diffirently by picking up diff provider for it
9. If we get subsequent failure, System would push them into a DLQ
10. We'll connect the db to an analytics store for analytics over click through and other stuff

## Datastore
1. Dynamodb for event store
2. Dynamodb for notification store
3. Dynamodb for notification config store

## Architecture diagram

![Architecture diagram](./assets/notification-system.drawio.svg "Architecure diagram")

## Components
1. Notification Consumer
2. Notification Queue
3. Notification Processor
4. Dedup later
5. Ratelimiter
6. SMS processor
7. Email processor
8. Whatsapp Processor
9. Push processor

## Flows

1. Notification flow
    - Service calls the notification event capture API
    - System captures the event into the notification queue
    - Notification processor fetches event from the queue
    - Notification processor checks for dedup basis event id
    - Ratelimiter is invoked and system checks if we can push the notification to the user
    - Notification processor then fetches the notification config
    - Basis the config the target channels are decided
    - Notification event for each channel is created and pushed into respective queue
    - Channel service consumes the event from the queue
    - Channel service forms the notification object basis the template
    - Channel service sends the notification to the provider
    - If either the send fails or the provider sends back a failure webhook callback system pushes the event into the failure queue
    - Failure messages are dealt with different provider
    - if a failure message fails again it is moved to DLQ for analysis and debugging

