# Leetcode 

## Requirements
Core Requirements
- User should be able to list down problems
- User should be able to view a problem
- User should be able to submit a solution in any allowed language and get instant feedback
- User should be able to attend a contest
- User should be able to see a live leaderboard for a contest

## Non functional requirements
Core Requirements
- The system should prioritize availability over consistency.
- The system should support isolation and security when running user code.
- The system should return submission results within 5 seconds.
- The system should scale to support competitions with 100,000 users.

## Entities

- User
- Problem
- Solution
- Evaluation
- Contest
- Contest Score
- Leaderboard

## APIs
- GET /problem - list all problems
- GET /problem/{problemId} - get problem
- GET /solution/{id} - get solution by id
- GET /contest - get all the contests
- GET /contest/{contestId} - get contest by contest id
- GET /contest/{contestId}/leaderboard - get leaderboard for the given contest

## Observations and design highlights
1. Since problems are mostly static and rarely modified, we can use single leader replicated RDBMS
2. We'll keep the solutions in an object store
3. Solutions would be executed in shell jail so that the code cannot access any system command
4. Solutions will first go through a static analysis for threat analysis
5. Solutions should be buffered in a queue and some dedicated workers will keep evaluating the solutions
6. Contests would have a dedicated queue and workers, so that the evaluation time is as low as possible
7. Workers would update the user score if evaluation is done for a contest
8. CDC from the score table would be picked by the leaderboard service which will internally update the in mem heap of the user scores
9. Every few seconds we'll get a lock on the heap and get a snapshot
10. All the leaderboard requests would be served the last captured snapshot
11. Since the leaderboard is in mem, we'll have few replicas also cosuming the same kafka queue, if node crashes follower would be picked as the master
12. Once a new node is booted, it will pick the last snapshot and consume all the events post that timestamp
13. We'll keep test cases in object store and will keep the test cases locally cached during the contest

## Datastore
1. RDBMS for problems table
2. Object store for the problem statements
2. Object store for the solution file
3. Dynamodb for solution table
4. Dynamodb for evaluation table
5. Dynamodb for contest table
6. RDBMS for contest score - sharded on contest id
7. Leaderboard in memory heap with periodic snapshots

## Architecture diagram

![Architecture diagram](./assets/leetcode.drawio.svg "Architecure diagram")

## Components
1. Problem service
2. Solution service
3. Evaluation workers
4. Leaderboard service

## Flows

1. User solving problem in contest
    - User logs in
    - User hits the contest link
    - System fetches the problems
    - System renders the problem once user opens the problem view
    - User submits the solution
    - client fetches for a presigned solution bucket link
    - client uploads the solution file to the object store
    - client then hits the solution service endpoint to capture the solution
    - Solution service puts the solution id and the link in the eval queue
    - Eval workers pick up the solution
    - Eval workers either fetches the TC from local cache or from TC object store, then run the test cases evaluation
    - Eval workers then save the evaluation into the eval table
    - CDC from eval table is captured into the score table
    - CDC from score table is captured by the leaderboard service
    - Leaderboard service updates the in mem heap for the user score
    - Leaderboard service takes a snapshot every few seconds/mins
2. User fetching leaderboard
    - User connects to the system
    - User open the leaderboard view
    - Leaderboard service returns the last captured snapshot

