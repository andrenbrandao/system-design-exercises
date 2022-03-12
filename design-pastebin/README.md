# Design Pastebin

Let's design a Pastebin like web service, where users can store plain text. Users of the service will enter a piece of text and get a randomly generated URL to access it.

## Step 1: Requirements Clarifications

- Are users going to be able to paste text only or also images?
- Is there a limit to how big the texts or images can be?
- Do users need to login or can they paste it anonymously?
- Should URLs expire after a certain period? How long after? Can the user choose it?
- Can logged-in users see statistics for their links? How many clicks, etc?
- Should the system be highly available? Or can we accept users sometimes not being able to access their data?
- Does it need to be highly reliable? That is, do we guarantee that uploaded data won't be lost?

### Functional Requirements

- Users will be able to paste data and get a unique URL.
- Users will only be able to paste text.
- Users can only paste text up to 5MB.
- They can do it anonymously. So we don't need to cover authentication here.
- Links will expire after a period of time. The default is 10 minutes. Users can also choose how long it will take.

### Non-Functional Requirements

- The system should be highly available. If nodes are down, users will still be able to access their pastes.
- It should also be highly reliable. We guarantee that data will not be lost.
- The servers should respond to requests with minimum latency.

## Step 2: Back-of-the-envelope Estimation

Scale/Traffic: read/write ratio.
Storage: how much?
Bandwith: what is the incoming and outgoing data?
Memory: will we need cache?

The system will be read heavy. Writes will be much less frequent.
Let's assume the Read:Write ratio is 10:1.

**Traffic Estimates:** Our system is supposed to receive 1 million pastes every day. So,
We will have 10 million reads/day.

If it is evenly distributed over the day, that would be 10.000.000 / (24*60*60) = ~120 req/sec for read requests and ~12 req/sec for writes.

| | |
|---|---|
|Read Requests|~120req/sec|
|Write Requests|~12req/sec|

**Storage Estimates:** The maximum size of text we will be able to handle is 5MB. But the average
text will be much less than that. Assuming that every character takes 1 Byte and that our users
will paste an average of 1000 words per paste, every write will have a size of ~10KB.

Every day, we will have:

Powers of Two Table

|2^n|Bytes|10^n|
|---|---|---|
|2^10|1KB|10^3|
|2^20|1MB|10^6|
|2^30|1GB|10^9|
|2^40|1TB|10^12|

1 million pastes \* 10KB = 10^6 \* 10KB = 10^7KB / 10^6 = 10GB.

So, we will have to store 10GB per day. How long do we need to store these pastes for?
Let's say that users will be able to set the paste to never expire and that only 10%
of them will choose this option. The rest of the pastes will be stored for 10 minutes and then discarded.

If we want to store this data for 5 years:
10% of 10GB per day \* 5 years = 1GB \* 365 \* 5 = **~2TB in 5 years**.
The rest of the 9 GB Storage would be discarded, since they would only last for 10 minutes.

We also need to take into consideration that we will need to generate keys to uniquely
identify the pastes. As we have 100K pastes per day that last for 5 years, we will need
200 million keys. If we use base64 encoding (A-Z, a-z, 0-9, .,-) we will need 5 letter strings:

64^4.6 ~= 200 million keys. So we can roundup to 5 leters: 64^5 ~= 1 billion keys.

If it takes 1 byte to store each character, the total size to store all keys would be:

1B * 5 = 5B Bytes = 5\*10^9 = **5 GB**

To keep some margin, let's assume a 70% capacity model, where we won't use more than 70%
of the available storage. Which raises our storage needs to **3TB**.

|||
|--|--|
|Storage|3TB for 5 years|

**Bandwith Estimates:** For write requests we expect 12 req/sec, and given every write is about
10KB, we have an ingress of 120KB/sec.

For read requests, we have 120 req/sec, giving an egress of 1200KB/sec = 1.2MB/sec.

|||
|--|--|
|Ingress|120KB/sec|
|Egress|1.2MB/sec|

**Memory Estimates:** We can cache the most accessed pastes. If we follow the pareto principle,
20% of our pastes will account for 80% of the requests. So, let's cache these 20%.

With 10M reads in a day, we would need to cache:

0.2 \* 10M \* 10KB ~= 20\*10^6\*10^3 ~= 20\*10^9 ~= 20GB

Also, notice that since most of our requests will be to expired pastes, because they
only last for 10 minutes, we will be able to remove them from the cache, decreasing this size.

|||
|--|--|
|Memory|20GB for Cache|

### Summary

|Type|Estimate|
|---|---|
|New Pastes|1M pastes/day|
|Reads|10M/day|
|Write Requests|120req/sec|
|Read Requests|12req/sec|
|Storage|3TB in 5 years|
|Incoming Data|120KB/sec|
|Outgoing Data|1.2MB/sec|
|Memory for Cache|20GB|

## Step 3: API Design

Given our requirements, let's create our API. User's should be able to paste texts, se let's create an endpoint for that.

`postText(text, expiration_period=10)`

**Request Parameters:**

- text (string): what the user wants to paste.
- expiration_period (integer): set in minutes. The default is 10. To never expire, set to -1.

**Returns: (string)**

- Generated URL with key.

`readText(key)`

**Request Parameters:**

- key: identifier of the pasted text.

**Returns:**

- The pasted text

### How can we prevent malicious users from abusing our APIs?

- We could use a rate limiter in our Load Balancers and block IPs that are making too many requests in a period of time.
- If we get too many requests from a IP Address, and they try to store big amount of texts? We could store the IP Addresses of the requests and later we could find which data came from that user. That way it would be easier to delete them.
- Also, if we could add authentication and require users to Login if we identify any malicious usage from their IPs.

## Step 4: Defining a Data Model
