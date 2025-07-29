# Pastebin

## Requirements
- User should be able to create a paste, which is essentially a blob, and get a link in return
- If user hits the link, he/she should be shown the paste
- configurable TTLs for each paste
- owner should be able to edit the paste
- System should support public / private pastes
- public ones should be accessible for all
- for private pastes only certain privileged users would have access which owner can configure

## Non functional requirements
- 10 Billion pastes 
- Paste read path should be fast
- very high throughput for read path
- should support high throughput for write path as well

## APIs
- POST /paste
- GET /paste/{id}
- PATCH /paste/{id}

### Scale

| Metric                            | Value                             | Notes                                           |
| --------------------------------- | --------------------------------- | ----------------------------------------------- |
| **Reads**                         | 100M - 1B/day (≈1.2K - 11.5K/sec) | Majority of the load.                           |
| **Writes**                        | 1M - 100M/day (≈11 - 1000/sec)    | Much lower than reads.                          |
| **Storage (Active pastes)**       | 10 B                               | Assume growing over years.                      |
| **URL Lifetime**                  | ∞ or expiring (1 week - 1 year)   | Some URLs can expire, others live indefinitely. |
| **Latency**                       | < 100ms for reads                 | Fast reads is key.                        |
| **Availability**                  | 99.99% or better                  | Mission-critical service.                       |
| **URL length (short)**            | 6-10 characters (base62/base64)   | Covers up to trillions of URLs.                 |
| **User base**                     | 100M+ users                       | Depends on business assumptions.                |

### Back of the envelop calculations
- read req rate - 10k - 20k per second
- writes - 1k per second
- storage - 640 GB overall
- number of short urls available with 8 bytes - 0-9 + a-z + A-Z = 62 possibilities for each char - 62 ^ 8 - 2.1834011e+14 which is 200 trillion 
- assuming 10% urls to be hot spots, their storage would be - 64 GB - so we can serve these from Redis or Memcache

### Schema

Paste
1. id - 8 bytes
2. title - 32 bytes
3. short url - 8 bytes
4. blob storage id - 8 bytes
5. ttl  - 8 bytes

ACL
1. id - 8 bytes
2. paste id - 8 bytes
3. user id - 8 bytes
4. status - 1 bit

Total storage for a short url - 64 bytes
Total storage per day - 64 bytes * 100 M - 6.4 GB per day
Total storage for overall - 64 bytes * 10 Billion - 640 GB

## Notes
1. Since the write requirements are not huge so essentially we only have to care about the reads
2. In order to serve the reads fast we should be redirecting them through in memory structure ie a read through cache
3. The fallback DB lookup for a short url should also be fast, essentially we can btree index on postgres for faster lookup on shorturl
4. Another problem would be generating the short urls there are many ways to tackle them like the following
    - Random short url generation, this should be fine for starting out when chances for collisions are low, once we start getting collisions it would be better to opt for other ways
    - Keep a single counter in memory or in DB which can tell use the next number to be used, which will further be encoded as per the encoding rules, this would become a single point of failure and a bottleneck if we want to scale the writes further.
    - Another way to further optimise this would be to keep the short urls pregenerated in the db and as an when the requests come, it can acquire one from the generated ones, this would make sure that there is no contention as no lock would be required.
    - Snowflake style id generation, this is a good read on how we can generate unique numbers globally without any DB - lets use this method

## Database
1. Redis for serving hot reads top 10% - 64 GB for pastes
2. Postgres for short url to object storage mapping - with btree index on shorturl
3. btree index on ttl
4. for ACL we'll have a combined b tree index on paste id and user id so that query if given user has access to this paste are fast

## Architecture diagram

![Architecture diagram](./assets/pastebin.drawio.svg "Architecure diagram")

## Flows

1. Paste creation flow
    - User hits the service with the paste data
    - Service creates a short url using snowflake style id creation
    - Service uploads the paste into blob storage
    - Service writes this back into the paste table
2. User hits the short url
    - Service first checks the Redis for the short url
    - Service then fetches the short url from postgres using the index
    - Service writes back the entry in Redis
    - Service makes another postrges index look up to see if the user has access to the paste
    - Service returns the presigned blob url
    - Client will fetch the paste from blob storage and render
    - Service then writes this event into kafka
3. Expiring the pastes
    - cron job reads the short urls expiring today from ddb using a paginated query
    - cron job audits these shorts urls into a kafka table
    - job then drops those urls
 
5. Analytics pipeline
    - Flink job keeps reading the paste read events from the kafka queue, aggregates them for sometime then writes them into the warehouse
    - Another flink keeps reading the paste drop topic and this into warehouse for analytics

Further notes:
- We can also keep some hot urls in a LRU style in mem on the servers itself to reduce the load on redis for popular urls
- We can also opt to use bloom filter before hitting the Redis, this would reduce a lot of cache miss and it would only require constant mem



