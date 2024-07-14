### URL Shortner
 
 #### Questions: 
 
 1. what is the length of the URL - output?
 2. Traffic?
 3. Single Instance System ? or need to be scaled
 
 #### Assumptions:
 
 1. Let's take 7 char as the post domain length in URL output 
 domain.com/<7char>
 2. Traffic - 30 Mil uses / month
 
 #### Data Capacity Model
 
 How much data we need to store in DB for 1 / 5 year etc, 
 For this attributes decided are:
 
 1. Long URL - input             - 2KB
 2. Short URL - output mapped    - 17 Byte
 3. createdAt - timestamp        - 7 Byte
 4. exipreAt - timestamp         - 7 Byte
 
 So each doc would have around 2.031 KB of data
 
 So for 30 Mil uses / month we would have:
 1. 60.7 GB / month
 2. 0.7 TB / year
 3. 3.6 TB / 5 year
 
 Now based on this we can deicde what kind of database should we use in our system design
 
 #### Hashing Algorithm
 
 We have multiple hashing algorithms for our usecases in real world. Let's discuss two of the most commonly used hash algos & decide which one is best for our usecase.
 
 1. MD5 Hash Algo
 
     This was one of the most commonly used Hashing Algorithms until hash collisions were proved in it.
 
     HashCollisions - when the output hash for two different inputs are same 
 
     Even if we ignore hash collision, the output string from MD5 is 128-bit message and we just need 7 characters length of URL shortened and if we decide to just take the 1st 7 characters of the output of MD5's 128 bit message, it'll further increases chances of hash collision and data correuption in our DB
 
 2. Base62 (B62)
 
     This is another hashing algorithm which uses 62 characters to encode a particular input - 
     
     62 Chars :- A to Z + a to z + 0 to 9
 
     62^7 (no.s of chars we need) ~ 3.5 Trillion unique Hashes
 
     If we go top use base10 (B10) instead consisting of just 0 to 9 then 10^ 7 ~ only around 10 million unqiue hashes, which doesn't meet our criteria for even a month with 30 Mil uses / month
 
 
 def base62_encode(encode):
     s = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'
     hashed_output = ''
 
     while encode > 0:
         hash_str = s[encode % 62] + hash_str
         encode /= 62
     
     return hash_str
 
 
 #### What Database to use ?
 
 1. RDBMS
     This is good for maintaining ACID properties, but issue comes in scaling this horizontally. 
 
     At per our traffic & data storage requirements , RDBMS usage would increase complexity
 
 2. NoSQL
     Good for scaling horizontally 
 
     Eventual Consistency -> Makes copies in different node , so some overhead here as well (acceptable tradeoff)
 
     Pros - Availability & Scalibility 
 
 
 #### Techniques to insert the solution doc (which includes Short URL) in the DB - 
 
 We need to handle a couple of edge cases while we insert a long-short URL mapping into the DB so in case of hash collision we don't corrupt the data in our DB.
 
 1. At each mapping / insertion we need to check if the short URL generated via our hashing algorithm like B62 is not already present in the DB 
 
 2. This again becomes complex when you have your app server running on multiple threads which might cause a race condition type of situation in which two nodes may have different input of long URL but returning same short URL and upon checking in the DB , they get no earlier instances of that short URL since the invocation in both nodes are done at the same time.
 
 3. For handling this we need to add logic at a DB insertion level - either make shortURL column a unique key (can be done in NoSQL as well) or something else. And then add logic if the insertion fails due to this, some other random hash needs to be generated.
 
 4. Same issues around MD5 algorithm incase of hash-collisions again check at a DB insertion level.
 
 5. The better approach with fail proof solution could be the following:
 
     ##### Better Approach - Use of Counter / Some kind of salting
 
     Keeping adding a prefix / postfix & increase the counter of that value at each new insertion in the DB , in this case even if: Two different long URLs being ran on the logic at the same time would have different INPUT itself with a salted value counter.
 
     This would make sure to avoid DB hash collisions / race conditions.
 
     Problem would come in scaling as you can't maintain a counter for all the nodes if the app is multithreaded.
 
     Even if try to centralize your counter which will give all your nodes access to the same counter, you're making your system vulnerable to a single point of failure in the case of the Counter being down at any point of time.
 
     Even if you keep different counters regionwise of seperate it via a range , the issue arises in co-existing when the counter runs out of the defined range.
 
     All in all the scalibility becomes unecessarily complex.
 
 
 #### Zookeeper - Apache Foundation
 
 This will help us tackle the issue with scaling.
 
 Zookeeper is a distributed coordination service, to manage multiple distributed hosts.
 
 1. Elects whos the leader / master
 2. Keeps configuration 
 3. Coordinates with the DB
 4. Keeps a health check on all node instances (for down time)
 
 By this race condition & everything is solved.
 
 Zookeeper will help with allocation and de-allocation of range of counters for each node seperately , so that no two nodes are working on the same range.
 
 ![](https://shorturl.at/x57J5 "Design")