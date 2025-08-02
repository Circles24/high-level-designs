# Social media 

## Requirements
- User should be able to follow someone
- Once the user is following someone, his/her posts should be visible in users feed
- User should be able to post
- User should be able to search users / posts
- User should be able to view who all he is following
- User should be able to view who all are following him

## Non functional requirements
- Monthly Active Users (MAU): 100 million
- Daily Active Users (DAU): 10 million
- Peak QPS (queries per second): 50,000 for read (feeds/search), 5,000 for write (posts/comments)
- Average posts per user/day: 2
- Post size (text, images, metadata): 1 KB (text only) to ~2 MB (with images)
- Feed latency target: < 300ms
- Search latency target: < 500ms
- Average person follows: ~40 people - 4 billion follower followee relationship

## APIs
- POST /post
- GET /post/user/{user-id}
- GET /user/profile/{user-id}
- GET /user/feed/{user-id}

### Scale
- Post create - 5k RPS
- Feed read - 50k RPS
- Search - 50k RPS

### Back of the envelop calculations
Storage Estimation
Posts
- Total posts/day: 10M users * 2 posts = 20M posts/day
- Post size average: ~500 KB (assume mixed media/text)
- Storage/day: 20M * 500 KB = ~10 TB/day
- Storage/year: 10 TB * 365 ≈ 3.6 PB/year

User Data
- Profile, settings, preferences: ~10 KB/user
- 100M users → ~1 TB

Follower data
- 16 bytes for a row * 4 Billion rows: 64 GB, we can keep this in redis or somewhere for caching as well

### Schema

User
1. id
2. name
3. gender
4. email
5. ...

Post
1. id - 8 bytes
2. user id - 8 bytes
3. title - 32 bytes
4. description - 80 bytes
5. photo urls - 100 bytes
6. ...

User Following
1. user id - 8 bytes
2. following id - 8 bytes

User Followers
1. user id - 8 bytes
2. follower id - 8 bytes

Feed entry
1. user id
2. post id
3. status
4. ...

## Notes
1. The most common operation for a social media is the feed, hence the feed fetching should be highly optimised
2. When a user posts something, we can take sometime to fan out it to all the users who are following him
3. For popular users, who's followers count is in million, it doesnt make sense to fanout their post, rather when reading the feed their posts should be explicitly queried
4. For search queries we can utilize an inverted index like elastic search with post index and user index
5. Once a post is created we can use change data capture to move it to elastic search cluster
6. We can utlise change data capture on users and profile table to move it to elastic search cluster

## Database
1. Cassandra for followers table with partition on user id, sort by following id, change data capture to build User following table in async 
2. Cassandra for posts table, partition by user id, sort id to be creation timestamp
3. Cassandra table for feed, partioned by user id, sort id to be creation timestamp
4. Followers data should be cached in redis and in mem in Flink nodes for fast lookup when fanning out posts
5. Postgres for users and profile data, here the query pattern is just read by primary key, which would be fast with postgres

## Architecture diagram

![Architecture diagram](./assets/social-media.drawio.svg "Architecure diagram")

## Flows

1. Post creation flow
    - Client fetches the presigned urls for object store
    - Service provides the presigned urls
    - Client uploads the media on the object store
    - Client hits the post creation endpoint
    - System stores it in the cassandra table with multi leader replication
    - Change data capture flow picks up the post
    - Flink node processing this is shareded on the user id
    - If the user is popular system drops the fannout
    - Else system fetches the following table for the user from the local state of flink (which is again filled from the redis)
    - For all the followers system enters the post id into the feed table of the cassandra
    - If the flink node crashes and picks the same post again we can dedup at the feed table layer for user id + post id combo
    - So this flow should be resilient as long as the flink nodes are able to handle the throughput
    - CDC flow also triggers the elastic search ingestion flow and post es ingestion the post would be applicable for searches
2. Feed reads flow
    - Service receives the request to load feed for a user id
    - Service searches cassandra for user id and gets the top k posts
    - Since the cassanda is partitioned on user id and sorted by timestamp hence this query should be fast
    - System also query the posts from the popular users which current requestor is following
    - Since the list of pop user current user follows would be limited, hence the above query should also be fast
    - Service also returns the db cursor along the response
    - Client makes another call the the feed with the user id and cursor
    - System reads from that cursor and fetches pop post using a timestamp based filter
3. Follow flow
    - User clicks on the follow button
    - System enters the entry into the following table
    - Since we are utlising a write through cache, we'll also add this user into the redis ie user : all the users this user if following
    - CDC captures this change and adds it into user followers table, which is the picked up by cdc and consumed by the flink node partitioned on user id
5. Search flow
    - Service directly hits the elastic search which internally parses the query and gives the response
    - Since elastic search takes care of the distributed systems part of the things, we should'nt care about this that much

Further notes:
- The popular users problem is called Justin Bieber problem
- Popular posts can be mostly predetermined by checking the user followers count, top posts can be cached into redis cache
- Feed is read directly from the in mem cache servers


