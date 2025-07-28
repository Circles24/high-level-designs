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
- storage - 14 TB for 5 year
- number of short urls available with 8 bytes - 0-9 + a-z + A-Z = 62 possibilities for each char - 62 ^ 8 - 2.1834011e+14 which is 200 trillion 
- assuming 1% urls to be hot spots, their storage would be - 140 GB - so we can serve these from Redis or Memcache

### Schema

Short url
1. id - 8 bytes
2. short url - 8 bytes
3. long url - on avg - 56 bytes
4. ttl day - 8 bytes

Total storage for a short url - 80 bytes
Total storage per day - 80 bytes * 100 M - 8 GB per day
Total storage for 5 years - 5 * 365 * 8 GB - 14 TB

## Notes
1. Since the write requirements are not huge so essentially we only have to care about the redirects
2. In order to serve the redirects fast we should be redirecting them through in memory structure
3. The fallback DB lookup for a short url should also be fast, essentially we can utilise key value db for capturing this like dynamodb
4. Another problem would be generating the short urls there are many ways to tackle them like the following
    - Random short url generation, this should be fine for starting out when chances for collisions are low, once we start getting collisions it would be better to opt for other ways
    - Keep a single counter in memory or in DB which can tell use the next number to be used, which will further be encoded as per the encoding rules, this would become a single point of failure and a bottleneck if we want to scale the writes further.
    - Another way would be to split up ranges and assign them to different mysql shards so that service can connect to their own respective shard and generate the short url, this would scale better sice we dont have single point of failure and the choke point is also distributed among shards
    - Another way to further optimise this would be to keep the short urls pregenerated in the db shards and as an when the requests come, it can acquire one from the generated ones, this would make sure that there is no contention as no lock would be required, lets go ahead with this one.
    - Snowflake style id generation, this is a good read on how we can generate unique numbers globally without any DB

## Database choice
1. Redis for serving hot redirects top 1%
2. Dynamodb for short url to long url mapping - very good horizontally scalbility
3. GSI on ttl day to figure out the short urls expiring on a certain day


## Architecture diagram

![Architecture diagram](./assets/url-shortner.drawio.svg "Architecure diagram")

## Flows

1. Short url creation flow
    - User hits the service with the long url
    - Service picks a short url from the mysql shard which is precomputed
    - Service writes this back into a short url table
    - Change data capture eventually picks this from mysql (eventual consistency here)
    - This entry is written to dynamodb which takes care of redirection
2. User hits the short url
    - Service first checks the Redis for the short url
    - Service then fetches the short url from dynamodb using key loop up (key is the short url here)
    - Service returns the mapped long url
3. Short url precomputation 
    - cron job reads the number of available short urls ready to be picked up
    - if we already have more than some configured short urls in the db then do nothing
    - a cron job reads the last offset from the mysql shard
    - cron generates the short url by encoding that number representation
    - then cron job writes it back into the mysql, and increase the counter using a transaction
4. Expiring the short urls
    - cron job reads the short urls expiring today from ddb using a paginated query
    - this query should be fast since we'll be using global secondary index on the ttl day field
    - then write those short urls in a kafka topic
    - then start deleting those entries from ddb
    - a job picks these short urls from the kafka and injects it back into the mysql table for available short urls


Further notes:
- Dynamodb natively supports TTL, so we can utilise that and hook the mysql insertion logic into the change data capture stream
- We can also keep some hot urls in a LRU style in mem on the servers itself to reduce the load on redis for popular urls
- We can also opt to use bloom filter before hitting the Redis, this would reduce a lot of cache miss and it would only require constant mem



