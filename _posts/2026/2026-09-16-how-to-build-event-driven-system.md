---
tags: [architecture]
---

# How to design event-driven system

I applied jobs and asked about how will I build an event-driven system, manage rate-limit and failure.

The question

> How do you structure Node.js backend services and scheduled sync jobs to reliably consume third-party APIs while
> handling rate limits or downtime?  

I have aksed this similar question in 2 instance of interview, one in written and the other live. Personally I'm not 
satisfied with how I answered this in live-interview. Because there is lot components that I need to remember, consider
and feel like if I explain in detail, it will take too much time and opt-out to not mention it in detail.

But when asked in written-interview, I was able to answer it in detail and feel confident about my answer. 

This is why I think asking to explain too much detail during live-interview is not a good idea, as the candidate
sometime miss to explain something then we correct it or silent which ultimately made them feel uncomfortable and will
fails.

DuckDuckGo as example also realized this and not doing this kind of live interview, instead opt-in with test project.
Read more about it here [https://duckduckgo.com/how-we-hire](https://duckduckgo.com/how-we-hire)

## How to design it
There are 2 major things that we must aware for task that run in background like sync-job: 

1. What should happen when it fails?
2. How do we know if it fails?


Then there is additional edge case like rate-limit, server downtime, etc. But core issue need to be solved are still those 2.

To solve this architectural case, this is how we should design it.

## Retry logic
Wrap the execution with `retry` logic, if it fails, system will automatically retry it. But do not retry it forever or instantly.

Instead put delay between each retry. 
When to delay it, is depend on the condition, if we got HTTP header response `retry-after`, then use it as guidance when to retry. 
If not, use exponential backoff algorithm to calculate delay. e.g: first fail, delay 30 seconds, second fail, delay 60 seconds, etc.

Have limit on how many times it can retry, if it still fails afterwards, then mark it as failed and notify the dev team. 
In AWS Cloud or general queue/messaging system, we can use dead-letter queue to handle this case which then later pooled and send the notification.

This will cover the case 1 & 2 mentioned above.

## Manage collide
Now come another edge case, what if 2 sync job run at the same time, how to handle it?  
This usually happen when retry logic with delay above happen until it hit the next sync schedule.

Use the lock mechanism to prevent the resource mutation in progress. Something like distributed lock should works.
Additionally I would also put idempotency to ignore duplicate request.

## Sample code
I have created similar logic in gist github with PHP with retry & fix delay, it didn't have idempotency and distributed
lock as out of scope and managed by the caller, but it should give you an idea how to implement it and improve it for your use case.

Deadlock Retry PHP Function 
[https://gist.github.com/gusdecool/8b406ee603a98039fb6d6de7b1555544](https://gist.github.com/gusdecool/8b406ee603a98039fb6d6de7b1555544)
