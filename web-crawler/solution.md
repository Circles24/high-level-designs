# Social media 

## Requirements
- Input a set of seed URLs to start crawling from.
- Post every crawl analysis, these seed uls can be tweaked to include the most relevant and important websites
- Crawl web pages starting from seed URLs and follow hyperlinks recursively.
- Extract and store data (e.g., HTML content, metadata, links).
- Respect robots.txt rules and site-specific crawling constraints.
- Avoid duplicate URLs (don't crawl the same page multiple times).
- Support politeness policies (delay between hits to same domain).
- Handle different content types (HTML, PDFs, images).
- Allow URL filtering (domain-specific, filetype-specific, etc.).

## Non functional requirements
- Scalability: Must support crawling billions of web pages across the globe.
- Throughput: Must be able to crawl millions of pages per day or more.
- Latency: Low latency in processing and storing each crawled page.
- Fault Tolerance: Must handle failures (network, hardware, etc.) gracefully.
- Consistency: Should avoid duplicate crawling, and ensure accurate updates of page versions.
- Availability: Always-on system; minimal downtime.
- Extensibility: Support plugins for content parsing, new protocols, etc.
- Politeness: Respect robots.txt, domain-specific rate limits, and avoid overloading servers.
- Monitoring & Logging: Track crawler performance, errors, and coverage.
- Security: Handle malicious pages, malformed responses, etc.
- Geo-distribution: Deploy across multiple regions to optimize latency and reliability
- Must crawl frequently changing websites frequently
- Must crawl important websites frequently
- Must be able to crawl the web in a week

### Back of the envelop calculations
a. Web Size
- Estimated number of unique URLs: ~50-100 billion
- Number of active domains: ~250 million
- Daily new/updated pages: ~200-500 million

b. Storage
- Average page size: ~500 KB (HTML + resources)
- Storage for 50B pages = 50B * 500 KB = ~25 PB (compressed)
- Metadata (URL, crawl date, hash, links): ~100 bytes/page → 5 TB

c. Crawl Throughput
- Daily crawling target: ~500 million pages/day
- Pages/sec = 500M / (24×60×60) ≈ 5,800 pages/sec

d. Bandwidth
- 500M pages/day × 500 KB = ~250 TB/day
- Bandwidth required = 250 TB / day ≈ 2.4 Gbps

e. Distributed Workers
- Assuming 1 worker can handle ~20 pages/sec:
- Need ~600 workers (min) for crawl load

f. Queue Size
- Frontier URL queue: tens of billions of URLs in persistent storage (sharded, deduplicated)

### Schema

Crawl
1. id - 8 bytes
2. url - 100 bytes
3. domain - 30 bytes
3. content hash - 64 bytes
4. crawled date - 16 bytes
5. s3 link - 100 bytes

Domain config
1. id - 8 bytes
2. domain - 8 bytes
3. importance - 8 bytes
4. frequency - 8 bytes

Domain Robots config
    domain - 8 bytes
    config - 200 bytes

## Observations and design highlights
1. Since the system is responsible for crawling the entire web, we would need hundreds of servers fetching data from the web and persisting them into a high write thoughput object store and persistent store
2. Since each web page contains multiple links we get a graph of pages linked together, after fetching each page, we should extract out all the links and push them to process ahead
3. We cannot keep bombarding single server ie website for pages, so the server distributing urls to crawl needs to be smart when giving out the urls to crawl to spiders
4. Precomputing these batches and keeping them ready for spider to consume and report back would be better rather than polling for urls every sec, ie batching in 100s
5. Once spider confirms a batch is processed drop it from the queue, else after timeout assign it to someone else
6. Each crawl batch should not contain more than a single url for a domain else we might be bombaring that server with requests
7. Assigner should also check if the url adheres to robots.txt policies for the given domain
8. Post fetching we should be persisting the content into an object store for further processing and indexing
9. Bloom filter can help up check if certain url has been fetched in this crawl session
10. Seeder server can trigger the pipeline by feeding the seed urls to the system
11. Ticker system makes sure that we are processing the fast changes websites frequently by ingesting those urls frequently
12. We'll keep DLQ for failed messages which would be picked up by a scheduler which will take care of the backoff part
13. After certain retries we'll move these entries into a final DLQ, which wont be processed

## Datastore
1. Object store for web content
2. Cassandra for crawl details
3. Postgres for keeping the domain configurations
4. Redis to cache the domain details
5. Bloom filters for quick lookup by url for current crawl

## Architecture diagram

![Architecture diagram](./assets/web-crawler.drawio.svg "Architecure diagram")

## Components
1. Seeder
2. Ticker
3. Filter
4. Feeder
5. Spider
6. Parser

## Flows

1. Crawl flow 
    - If we want to kick off a new crawl session, Seeder injects seed urls into the suggestion layer
    - Filter system, checks if the url has already been crawled in this session
    - Filter system checks if the domain needs to be crawled at all basis last crawled date, importance and freq of updates
    - Filter matches the urls against the cache of Robots.txt so that crawl adheres to the robots.txt file
    - The urls that pass these rules are sent to the Feeder system
    - Feeder groups the urls per domain basis, so that only handful of these are passed to the spider for each consumption request
    - Feeder internally priortises domains, and makes the pre aggregated batches of these
    - Feeder internally organises these batches as per priority
    - When a spider asks for feed, feeder gives the highest priority batch
    - Feeder takes care of the making sure we aren't bombaring a single domain
    - Feeder also makes sure we are giving priority to imporant websites, score of which can be determined post crawl analysis
    - Spider collects a feed from the feeder
    - Spider divides it into mini batches
    - Spider starts fetching these urls
    - The endpoints which fail are sent back to a DLQ with added exp back off time and jitter
    - Spider writes the content of each page into an object store
    - Spider then makes an entry into the crawl Cassandra DB, which operates in leaderless mode with parition by domain
    - Spider pushes then an event into parser queue
    - Parser picks up the event, first it fetches the webpage from object store, then fetches all the nested urls from the webpage and pushes them into the suggestion queue
2. Ticker URL injection
    - Ticker keeps track of important websites which update frequently, it on a periodic basis keep injecting those freq changign urls into the suggestion queue


