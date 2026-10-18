# System Design: Typeahead (Autocomplete) Search
Notes based on the provided video transcript. The design focuses on autocomplete suggestions ranked by search frequency, not on implementing the full search engine that returns search results.

## 1. Problem Statement
Design a typeahead/autocomplete system that suggests the most popular search strings while a user types.
Examples:
- A user types budget 2025.
- The UI suggests budget 2025 date, budget 2025 income tax, budget 2025 highlights, and similar queries.
- The user selects a suggestion and the actual search results page opens.
This pattern appears in search on e-commerce, food delivery, social media, travel, booking, and general search platforms.

### Two distinct operations
1. Autocomplete query: Given a partial prefix, return the top K matching search strings.
2. Selected search query: When a user selects a suggestion and opens the results page, record that search so its popularity can influence future rankings.
The video deliberately scopes the problem to autocomplete rather than all possible search types, such as tag filtering or general keyword search.

## 2. Functional Requirements
1. Given a prefix, return suggestions that start with that prefix.
2. Return the top 10 suggestions by search frequency (K should be configurable).
3. Start suggesting only after the user has typed at least 3 characters, and stop after 20 characters (the example requirements used in the video).
4. Treat suggestions as lowercase/case-normalized strings.
5. When a user selects a suggestion and opens its results, increment the selected search query's frequency.

### Good-to-have, not required for the core interview design
- Personalized suggestions. The video excludes personalization from the core scope because it introduces a separate, more complex ML/personalization problem.

## 3. APIs

### API A — autocomplete
GET /api/v1/autocomplete?query=budget
Example response:

```json
{
  "success": true,
  "results": [
    "budget 2025 date",
    "budget 2025 income tax",
    "budget 2025 highlights"
  ]
}
```
The response is a list of search strings, not the actual search-results page.

### API B — record a selected search
Conceptually:
PUT /api/v1/increment-search
Example request:

```json
{
  "query": "budget 2025 date"
}
```
The video suggests returning a simple success/failure response rather than exposing the exact count.
Security note: This increment operation should not be an unrestricted public endpoint. Otherwise, bots could repeatedly increment a chosen query and artificially push it to the top. Trigger it through a trusted server-side flow, with appropriate authentication/validation and abuse protection.

## 4. Non-Functional Requirements and Design Choices

### Low latency
Autocomplete should feel instantaneous. The video proposes a target of under 10 ms for the autocomplete path. This is an ambitious target and should be treated as a design goal, not a guarantee.
The increment operation does not need to delay the search-results response; it can run asynchronously.

### Availability over immediate consistency
Exact, instantly synchronized counts are not essential. If a query's count increases by a few searches, the suggestion order does not need to change immediately.
- Strong/immediate consistency: every replica reflects an update immediately.
- Eventual consistency: replicas and derived rankings can catch up over time.
For this system, the video chooses availability and eventual consistency over immediate consistency. Users should still receive suggestions even if ranking data is slightly stale.
The system aims to show useful popular suggestions, not guarantee that every result is in perfectly exact frequency order at every instant.

### Small counting errors are acceptable
The transcript gives an example where 100,000 searches are recorded as 99,600. For approximate popularity ranking, losing a small number of increments may be acceptable. This assumption enables batching and sampling, but it would not be appropriate for every product or business metric.

### Peak traffic and resilience
The system should handle traffic surges, horizontally scale the autocomplete service, and avoid making the user wait for counting/ranking updates.

## 5. Scale Estimation
The video uses these illustrative assumptions:
Input	Assumption
Monthly active users (MAU)	100 million
Daily active users (DAU)	20% of MAU = 20 million
Searches per active user per day	50
Total searches per day	20M × 50 = 1 billion
Seconds per day	86,400 ≈ 100,000
Average actual-search QPS	1B / 100,000 ≈ 10,000 QPS
Autocomplete calls per search, after debouncing	Approx. 3–5
Estimated average autocomplete QPS	Approx. 30,000–50,000 QPS
Illustrative peak multiplier	20×
Illustrative peak autocomplete QPS	Up to approx. 1 million QPS

Important transcript correction: The transcript appears to say “a billion MAU” and then calculates 20 million DAU. Those figures conflict. The calculation of 20 million DAU at 20% implies 100 million MAU, which is what the table uses. The transcript also contains an arithmetic inconsistency in one step: 10,000 actual-search QPS multiplied by 3–5 autocomplete calls gives about 30,000–50,000 autocomplete QPS, not 50,000 in every case.
These numbers are interview assumptions, not measured traffic figures for Google or any specific company. State assumptions explicitly and refine them if the interviewer gives different inputs.

### Why actual searches are not the whole story
Autocomplete can call the backend repeatedly during one search. After the user has typed three characters, typing b, u, d, g, e, t, and so on may trigger multiple requests. Therefore, autocomplete QPS can be several times higher than completed-search QPS.

### Debouncing
Debouncing delays execution until the user has paused typing for a configured interval.
Without debouncing:
- The user types each character.
- The client may send a request for every character after the minimum prefix length.
With debouncing:
- The client waits for a short pause.
- It sends one request for the latest prefix instead of one request per keystroke.

### Benefits:
- Fewer requests and lower backend load.
- Less frequent UI updates.
- Better typing experience.
The pause duration is a product/performance trade-off. The transcript mentions tiny pauses as examples and also discusses longer pauses; it does not establish one universally correct duration.

### New queries and traffic spikes
The video also considers:
- A portion of daily queries being new (one example uses 50% of 20 million DAU = 10 million new queries/day, though the exact assumption is not rigorously established).
- Spikes during major news or trending events.
- A peak multiplier such as 20× average traffic.

## 6. Minimum Data to Store
At minimum, keep a mapping:
search_string -> search_count
Example:
Search string	Count
how to cook	11
how to kill your roommate	10
how to	9

The count represents how often the completed search query was selected/performed. It is used to rank autocomplete suggestions.
The system also needs a way to quickly map a prefix to its top K suggestions.

## 7. Core Algorithmic Problem
For an input prefix S:
1. Find all search strings that begin with S.
2. Rank those strings by search frequency.
3. Return the top K results.
This combines:
- Prefix search
- Top-K ranking
A data structure that solves prefix matching alone does not automatically solve top-K ranking efficiently.

## 8. Why Not Simply Use a Trie?
A trie (prefix tree) stores strings character by character and shares common prefixes.
For example, apple and apps share the path a → p → p, then branch into l → e and s.
(root)
  └── a
      └── p
          └── p
              ├── l ── e   (apple)
              └── s        (apps)

### Benefits
- Prefixes are naturally represented.
- Common prefixes share nodes.
- Prefix lookup is efficient.
- Insertions do not require shifting an entire sorted list.

### Limitations for this system
1. A basic trie finds matching strings but does not automatically return the most frequent top K.
2. A straightforward approach finds the prefix node, performs DFS over its subtree, collects all matching terms, sorts by frequency, and returns the top K. This can be expensive for common prefixes.
3. The design still needs persistence, replication, sharding, and operational reliability. The video argues against building a custom persistent trie database for this interview design.
4. Sharding a trie while efficiently computing global top-K suggestions adds complexity.
5. Storing popularity information at every node can help, but it increases memory usage and requires updates along prefixes.
The video mentions that a sorted list plus binary search is another possible prefix-search approach, but maintaining sorted order can make insertions costly. Its main point is that a trie is not the complete answer to the distributed system problem.

## 9. Optimization: Precompute Top K for Every Prefix
Instead of running DFS and sorting all matching terms for every request, store the top K suggestions for each prefix in advance.
Example:
Prefix "h"  -> [Harry Potter: 30, Hollywood: 20, ...]
Prefix "ho" -> [Hollywood: 20, How to kill your roommate: 10, ...]
Prefix "how" -> [How to kill your roommate: 10, How to: 9, ...]
When a request for how arrives, read the precomputed list instead of traversing the entire subtree.

### Advantages
- Much faster autocomplete reads.
- Avoids repeated DFS and sorting at request time.
- Uses extra memory to reduce query latency.

### Trade-off
The precomputed lists become derived data that must be updated when counts change. The design accepts some delay in updates, so suggestions can be slightly stale.
This is a classic system design trade-off: spend more storage and update work to make reads faster.

## 10. Two Logical Mappings
The video describes two separate logical data structures.

### A. Frequency mapping
search string -> total count
Example:
"how to cook" -> 11
"how to kill your roommate" -> 10
This is the central count for each complete search term.

### B. Prefix mapping
prefix -> top K autocomplete suggestions
Example:
"how" -> [("how to cook", 11), ("how to kill your roommate", 10), ...]
"how t" -> [("how to cook", 11), ("how to kill your roommate", 10), ...]
When a completed search query is counted:
1. Increment its frequency.
2. Generate its prefixes.
3. Update the top-K lists for those prefixes if the query belongs in them.
A single query can affect many prefixes. For example, how to cook has prefixes such as h, ho, how, how , how t, and so on.
The video suggests maintaining each prefix's ranked suggestions using a suitable structure such as a hash map plus heap, a sorted set, or another ordered collection.

## 11. Why Redis Sorted Sets?
The video proposes Redis as an in-memory key-value store and uses its sorted set data structure for ranked suggestions.
A sorted set stores unique members with associated scores and keeps them ordered by score. In this design:
- Member: a search string.
- Score: its frequency/count.
This is a natural fit for a leaderboard-like operation: retrieve the highest-ranked suggestions for a prefix.
Conceptually:
prefix:"how" -> sorted set of (search term, frequency)
prefix:"how t" -> sorted set of (search term, frequency)
The frequency mapping is stored separately from the prefix-to-top-K mapping.

### Why use a managed, established data store?
Compared with building a custom hash map plus tree/set from scratch, Redis offers existing data structures, persistence options, and clustering/sharding capabilities. The exact latency and throughput depend on configuration, data size, network, and deployment; the video’s figures are illustrative rather than universal guarantees.

### A design detail to clarify in an interview
Redis sorted sets order by score, but the application must still define:
- How to handle equal scores.
- Whether only top K members are retained or all members are retained.
- How a changed score affects ordering.
- How to avoid stale/duplicate updates.

## 12. Reducing Writes: Batching and Sampling
Updating every prefix after every completed search can be expensive. The video proposes two ways to reduce update work.

### Strategy A — Batch/threshold updates
Do not refresh the prefix ranking for every single count increment. Refresh when the count has changed enough, or when a batch of updates is ready.
Example:
- A search term has a count of 1,000.
- The count increases with new searches.
- The system accumulates increments and refreshes the prefix rankings after a threshold, such as 100 increments.
This reduces how frequently the derived prefix index must be updated.

### Strategy B — Sampling
Update ranking data for only a random sample of search events, such as 1 in 100 or 1 in 1,000.
The intuition is that, at very large scale, a representative random sample may preserve broad popularity trends even if it does not preserve exact counts.
Trade-off: Sampling reduces write load, CPU work, and memory/update pressure, but introduces statistical error. It can miss rare or newly trending terms, especially before enough samples accumulate. A production design would need to choose the sampling policy carefully and potentially treat emerging trends separately.
Batching and sampling are alternatives or can be combined, depending on accuracy and load requirements.

## 13. High-Level Architecture
```text
                         AUTOCOMPLETE READ PATH

### Client
  |
  | debounced prefix query
  v
API Gateway (rate limiting / request controls)
  |
  v
Horizontally scaled Autocomplete Service
  |
  | read top K for prefix
  v
Redis / distributed cache
  |
  v
Return suggestions to client

                         SEARCH / UPDATE PATH
Client selects suggestion
  |
  v

### Search Results Service
  | \
  |  \ returns paginated search results
  |   ---------------------------------> Client
  |
  | asynchronously emits selected-search event
  v
Message Queue (e.g. SQS; Kafka may suit higher durability/throughput needs)
  |
  v
Background Data Processor
  |
  +--> update search frequency mapping
  |
  +--> update prefix-to-top-K ranking data
       in Redis / distributed cache
```

### Component responsibilities

### Client
- Debounces typing.
- Sends the latest prefix after a pause.
- Displays suggestions.
- Sends a completed search when a suggestion is selected.

### API Gateway
- Applies rate limiting and request controls.
- Provides a common entry point to the service.

### Autocomplete Service
- Horizontally scales to handle read traffic.
- Reads the precomputed top-K suggestions for a prefix.
- Returns suggestions quickly.

### Search Results Service
- Serves the actual search-results page, potentially paginated.
- Emits an event to record the selected search without making the user wait for ranking updates.

### Message queue
- Decouples the search response from the update pipeline.
- Buffers work so the processor can consume it asynchronously.
- SQS is the video's simple example; Kafka is mentioned as an option when throughput and durability requirements justify it.

### Background processor
- Consumes search events.
- Updates the frequency mapping.
- Updates the prefix ranking index, using batching or sampling where appropriate.

### Redis
- Stores the prefix-to-ranked-suggestions data and frequency mapping in the proposed design.
- Supports fast reads and ordered-set operations.
- Can be deployed in a sharded/distributed configuration.

### Optional analytics extension
The video mentions periodically exporting data to S3 and then loading it into a warehouse such as Redshift for analytics. This is optional and should not be added to the core design unless the interviewer asks for historical analytics or reporting.

## 14. Important Request Flows

### Flow A — User types a prefix
1. User types a few characters.
2. Client waits for the debounce interval.
3. Client sends an autocomplete request through the API Gateway.
4. Autocomplete Service reads the top-K list for that prefix from Redis.
5. Service returns suggestions.
6. Client displays them.
The read path avoids scanning every possible search term.

### Flow B — User selects a suggestion
1. User selects a suggestion.
2. Search Results Service returns the actual results page.
3. A selected-search event is emitted asynchronously.
4. A background processor consumes the event.
5. The frequency mapping is updated.
6. The prefix-to-top-K lists are updated immediately, in a batch, or according to the chosen sampling policy.
7. Future autocomplete requests eventually reflect the updated popularity.

## 15. Bottlenecks and Follow-Up Discussion

### What if autocomplete traffic grows?
- Horizontally scale the stateless Autocomplete Service.
- Use a distributed/sharded Redis deployment.
- Cache or replicate hot prefix data if needed.
- Apply rate limits and keep the client debounce behavior.

### What if a very common prefix is requested constantly?
Prefixes such as a, ho, or how can be hot keys. The minimum prefix length reduces the broadest searches, but hot prefixes can still be frequent. Consider caching popular prefix responses, replication/read scaling, and monitoring key-level traffic.

### What if ranking updates are delayed?
That is an accepted trade-off under eventual consistency. Make sure the queue and processor catch up, monitor queue lag, and define how quickly rankings should reflect new searches.

### What if the queue or processor is down?
The queue should buffer events within its retention limits. Monitor backlog and processing failures, retry transient failures, and route poison messages to a dead-letter queue where supported. The video mentions a queue and background processor but does not detail these operational recovery mechanisms.

### What if a search term is artificially boosted?
Do not expose an unrestricted count-increment API. Validate that the event corresponds to a legitimate selected search, protect endpoints, rate-limit suspicious activity, and monitor unusual query-count spikes.

### How do we handle new or trending queries?
Sampling can delay discovery of a new term. Possible extensions include higher sampling rates for new terms, a separate recent-trend signal, or periodically rebuilding rankings. These are follow-up ideas beyond the video's core proposal.

### How should data be sharded?
The video states that Redis can handle sharding based on keys. In an interview, clarify the key design and the consequence: prefix keys can be distributed, but a query update may touch many prefix keys. Ensure the chosen layout supports both fast prefix reads and efficient updates. The transcript does not fully specify the exact sharding key or cluster topology.

## 16. Interview Questions and Sample Answers
Q1. What exactly are we designing?
Answer: An autocomplete system that returns the top K search strings matching a typed prefix, ranked by completed-search frequency. We are not designing the entire search engine that retrieves the actual pages/products.
Q2. Why are there two different operations?
Answer: Autocomplete reads suggestions for a partial prefix. A completed search occurs when a user selects a suggestion or submits a query; that event changes the popularity counts used for future ranking.
Q3. Why is eventual consistency acceptable?
Answer: A small delay in popularity counts does not significantly harm the usefulness of suggestions. Availability and low latency are more important than immediately updating every replica for every increment.
Q4. Why debounce on the client?
Answer: Without debouncing, typing a single query can generate a request per character. Debouncing waits for a short pause and sends fewer requests, reducing backend load and unnecessary UI updates.
Q5. Why isn't a trie enough?
Answer: A trie is good for prefix matching, but a basic trie does not solve top-K-by-frequency ranking, persistence, distributed sharding, or update management. A DFS of all matches followed by sorting may be too expensive for common prefixes.
Q6. How do we make autocomplete reads fast?
Answer: Precompute the top-K suggestions for each prefix and store them in a fast key-value store. The request can retrieve the ranked list directly rather than traversing all matching terms.
Q7. Why Redis sorted sets?
Answer: The problem resembles a leaderboard. A sorted set associates each unique search string with a score, such as its count, and supports ordered queries. It avoids having to build and operate a custom ranking structure from scratch.
Q8. Why separate frequency data from prefix data?
Answer: One completed query has one central frequency, but it can affect many prefixes. Separating the count mapping from the prefix ranking index makes the responsibilities clearer and allows the ranking index to be updated asynchronously.
Q9. How do we reduce write load?
Answer: Batch updates and refresh rankings after a threshold, or use random sampling if approximate trends are sufficient. Both reduce ranking-update work at the cost of freshness or exactness.
Q10. Why use a queue?
Answer: It decouples serving the search-results page from updating popularity data. The user receives results without waiting for the ranking update, while a background processor handles the event.
Q11. What is the biggest trade-off in precomputing top K per prefix?
Answer: Reads become faster, but storage increases and updates must touch multiple prefixes. The design deliberately trades additional storage and asynchronous update work for lower read latency.
Q12. What is the difference between autocomplete and actual search results?
Answer: Autocomplete returns likely query strings while the user is typing. Actual search returns the results for a selected or submitted query and may use a completely different retrieval/ranking system.

## 17. Quick Revision Sheet
- Problem: Prefix autocomplete; return top K suggestions by search frequency.
- Minimum prefix length: 3 characters in the video's example; maximum 20.
- Main APIs: Read autocomplete suggestions; record a completed/selected search.
- Read path: Client debounce → API Gateway → Autocomplete Service → Redis → top-K suggestions.
- Update path: Selected search → Search Results Service → queue → background processor → frequency and prefix-ranking data.
- Core data: search_string -> count and prefix -> top K suggestions.
- Trie: Useful for prefix search, but not the complete distributed top-K solution.
- Read optimization: Precompute top-K lists for prefixes.
- Ranking structure: Redis sorted sets, with frequency as score.
- Consistency: Eventual consistency is acceptable.
- Load reduction: Client debouncing, batching, sampling.
- Scalability: Horizontally scaled service, distributed/sharded data store, queue-backed updates.
- Security: Do not expose an unrestricted count-increment endpoint.
- Main trade-off: Faster reads and greater availability in exchange for more storage and slightly stale/approximate rankings.

## 18. One-Minute Interview Summary
“I would design autocomplete as a low-latency read system with an asynchronous ranking-update pipeline. The client debounces typing and calls a horizontally scaled Autocomplete Service, which fetches a precomputed top-K list for the prefix from Redis. I would maintain a frequency mapping for complete search terms and a separate prefix-to-top-K index, using sorted sets to rank suggestions. When a user completes a search, the Search Results Service emits an event to a queue; background workers update the counts and prefix rankings. Because exact rankings do not need to change instantly, I would use eventual consistency and reduce write load with batching or sampling. This keeps the read path fast while allowing the system to scale.”

## Source and Scope Notes
These notes follow the supplied transcript's core design and terminology. The transcript contains a few inconsistent or ambiguous scale-estimation statements; the notes flag the main arithmetic inconsistency rather than treating every number as authoritative. Additional operational follow-ups are labeled as extensions rather than claims made by the speaker.
