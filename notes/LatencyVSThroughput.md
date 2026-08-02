
I mixed these up in a performance review meeting, in front of my director:

Which one would you optimize for first, and
why?
Latency VS Throughput


## 💡 The Big Picture & Analogy
Latency and Throughput are like "how fast one car travels" versus "how many cars total can pass through a road per hour" 
they are **not the same thing**, and sometimes improving one can actually make the other worse.

* **Latency** = the time a single request takes from being sent to getting a response back (measured in ms) — "how long does it take this one car to go from A to B?"
* **Throughput** = the number of requests a system can handle per unit of time (e.g., requests/second) — "how many cars can this road handle per hour?"

## 🧩 Why Do We Need It? (Why Separate These Two Clearly)
The common mistake is assuming "a fast system = a system that can handle heavy load," which **isn't always true**, because:

* A system with very low latency (responds super fast) might handle very few concurrent requests if poorly designed (e.g., single-threaded processing)
* A system with very high throughput might achieve that by **batching requests** before processing them together,
* which actually **increases** the latency of each individual request, since it now has to wait in a queue

In short, these two metrics are usually a **trade-off**, not something that improves together automatically.

## ⚙️ Core Logic & How It Works
Consider a simple example of a batch-processing system:

* If the system waits to collect 100 requests before processing them all at once (batch processing)
* → **Throughput is very high** because it processes a large volume per cycle, but the first request that arrives has to **wait** until the batch fills up → **its latency increases**
* Conversely, if the system processes each request the moment it arrives (no batching)
* → **Latency is low** (instant response), but **throughput might be lower** because resource usage isn't being optimized in bulk

## ⚖️ Trade-offs & When to Use

**When to prioritize Latency:**
* Systems where the user is directly waiting for a real-time response — e.g., web pages, search autocomplete, gaming, video calls — users feel it immediately if it's slow
* Systems with strict SLAs on response time — e.g., trading systems, payment gateways

**When to prioritize Throughput:**
* Background jobs where the user isn't waiting for a real-time result — e.g., batch data processing, ETL pipelines, sending bulk emails, log aggregation
* Systems that need to scale to large volumes and are measured by "how much work gets completed per day" rather than "how fast each individual job finishes"

**The core trade-off:** optimizing for latency usually costs more resources per request (e.g., caching, more servers, less batching), 
while optimizing for throughput usually costs higher per-request latency (because of batching or queuing).

## 🛠️ Real-World Scenario / Mini Example

**Context that changes the answer:**
* **In a user-facing API context** (e.g., mobile app checkout): "I'd optimize for **Latency first**, because the user is directly waiting for the response.
* Research from Amazon/Google has shown that even small increases in latency — every additional 100ms — measurably hurt conversion rates and revenue."
* 
* **In a data pipeline context** (e.g., a nightly job that summarizes daily sales): "I'd optimize for **Throughput first**,
* because the business doesn't care how many milliseconds each individual record takes to process
* what matters is whether the entire pipeline can finish processing millions of records within the required time window (e.g., must complete before 6 AM)."

## 🧠 Lead's Key Takeaway
**Golden Rule:** Never answer "which one I'd optimize first" without context 
the best response is to **ask back first: "Is the user waiting for this result in real time?"** 
If yes → prioritize Latency. If it's a background job measured by total work completed → prioritize Throughput. 
And always add that the two usually trade off against each other — you rarely get to optimize both for free at the same time.
