# URL shortner

## Requirements
- User should be able to paste a long url and get a short one in return
- If user hits the short url, he/she should be redirected to the original url
- configurable TTLs for each short url
- analytics on top of each short url redirection for billing

## Non functional requirements
- 1 Billion short urls 
- redirects to be as fast as possible
- very high throughput for redirections
- should support high throughput for creations as well

## APIs
- POST /url/create
- GET /url/{id}
- GET /short-url/{url}

### Scale

| Metric                            | Value                             | Notes                                           |
| --------------------------------- | --------------------------------- | ----------------------------------------------- |
| **Reads (Redirection requests)**  | 100M - 1B/day (≈1.2K - 11.5K/sec) | Majority of the load.                           |
| **Writes (Shorten URL requests)** | 1M - 100M/day (≈11 - 1000/sec)    | Much lower than reads.                          |
| **Storage (Active URLs)**         | 10B total URLs                    | Assume growing over years.                      |
| **URL Lifetime**                  | ∞ or expiring (1 week - 1 year)   | Some URLs can expire, others live indefinitely. |
| **Latency**                       | < 100ms for redirection           | Fast redirection is key.                        |
| **Availability**                  | 99.99% or better                  | Mission-critical service.                       |
| **URL length (short)**            | 6-10 characters (base62/base64)   | Covers up to trillions of URLs.                 |
| **User base**                     | 100M+ users                       | Depends on business assumptions.                |

### Back of the envelop calculations
- redirection req rate - 10k - 20k per second
- writes - 1k per second
- storage - 1 TB overall
- number of short urls available with 8 bytes - 0-9 + a-z + A-Z = 62 possibilities for each char - 62 ^ 8 - 2.1834011e+14 which is 200 trillion 
- assuming 10% urls to be hot spots, their storage would be - 100 GB - so we can serve these from Redis or Memcache

### Schema

Short url
1. id - 8 bytes
2. short url - 8 bytes
3. long url - on avg - 80 bytes
4. ttl day - 8 bytes

Total storage for a short url - 104 bytes
Total storage per day - 104 bytes * 100 M - 8 GB per day
Total storage for overall - 104 bytes * 10 Billion - 1 TB

## Notes
1. Since the write requirements are not huge so essentially we only have to care about the redirects ie the reads
2. In order to serve the redirects fast we should be redirecting them through in memory structure ie a read through cache
3. The fallback DB lookup for a short url should also be fast, essentially we can btree index on postgres for faster lookup on shorturl
4. Another problem would be generating the short urls there are many ways to tackle them like the following
    - Random short url generation, this should be fine for starting out when chances for collisions are low, once we start getting collisions it would be better to opt for other ways
    - Keep a single counter in memory or in DB which can tell use the next number to be used, which will further be encoded as per the encoding rules, this would become a single point of failure and a bottleneck if we want to scale the writes further.
    - Another way to further optimise this would be to keep the short urls pregenerated in the db and as an when the requests come, it can acquire one from the generated ones, this would make sure that there is no contention as no lock would be required, lets go ahead with this one.
    - Snowflake style id generation, this is a good read on how we can generate unique numbers globally without any DB

## Database
1. Redis for serving hot redirects top 10%
2. Postgres for short url to long url mapping - with btree index on shorturl
3. btree index on ttl

## Architecture diagram

![Architecture diagram](./assets/url-shortner.drawio.svg "Architecure diagram")

## Flows

1. Short url creation flow
    - User hits the service with the long url
    - Service picks a short url from the postgres which is precomputed
    - Service writes this back into the short url table
2. User hits the short url
    - Service first checks the Redis for the short url
    - Service then fetches the short url from postgres using the index
    - Service writes back the entry in Redis
    - Service returns the mapped long url
    - Service then writes this event into kafka
3. Short url precomputation 
    - cron job reads the number of available short urls ready to be picked up
    - if we already have more than some configured short urls in the db then do nothing
    - a cron job reads the last offset from the Postgres
    - cron generates the short url by encoding that number representation
    - then cron job writes it back into the Postgres, and increase the counter using a transaction
4. Expiring the short urls
    - cron job reads the short urls expiring today from ddb using a paginated query
    - cron job audits these shorts urls into a kafka table
    - job then drops those urls
 
5. Analytics pipeline
    - Flink job keeps reading the redirection events from the kafka queue, aggregates them for sometime then writes them into the warehouse
    - Another flink keeps reading the url drop topic and add the analytics for this

Further notes:
- We can also keep some hot urls in a LRU style in mem on the servers itself to reduce the load on redis for popular urls
- We can also opt to use bloom filter before hitting the Redis, this would reduce a lot of cache miss and it would only require constant mem



