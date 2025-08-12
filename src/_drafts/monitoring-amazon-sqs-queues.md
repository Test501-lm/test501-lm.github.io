---
layout: post
title: "Monitoring Amazon SQS Standard Queues"
date: 2025-08-04
categories:
---

## What is an Amazon SQS queue?

We'll skip right ahead to monitoring Amazon SQS queues {% cite awsDocsSqs %}, so
refer to the documentation
[here](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html)
if you need a refresher on how they work.

We're only going to consider standard queues here (including DLQs {% cite awsDocsSqsDlq %}),
not FIFO {% cite awsDocsSqsFifo %} or fair {% cite awsDocsSqsFair %} queues.

## What should you monitor?

We're going to consider the different metrics that are worth monitoring for your queue(s), and how we should
monitor them.
For each metric, we'll consider:
- the alerts to configure,
- how to display the metric in a "Health-at-a-glance" dashboard that we can
use to quickly check your system health,
- and how to display the metric in a more detailed dashboard for capacity planning and investigations.

### Age of the oldest message (almost)

_ApproximateAgeOfOldestMessage_ is the 'age of the oldest unprocessed message in the queue' {% cite awsDocsSqsMetrics %} - with a few surprises:
- **Poison-pill messages are ignored.**
  Regardless of whether or not a DLQ is configured, if a message hasn't been deleted after being received 3 or more times then it gets moved to the back of the queue and excluded from this metric {% cite awsDocsSqsMetrics %}.
- **DLQs report that messages are younger than they really are.**
  Messages that were automatically moved to a DLQ for exceeding the max receive count report the time since they were _moved_ to the DLQ for this metric {% cite awsDocsSqsMetrics %}.
  Unfortunately, the DLQ's retention period uses the time since these messages were sent to the _original_ queue to decide when to delete them {% cite awsDocsSqsDlqRetention %}.
  That means _ApproximateAgeOfOldestMessage_ could be far less than the retention period of the DLQ and messages are being deleted for being too old with you even realising!
  <!-- TODO: what about manual moves to the DLQ? -->
  (Redriving messages from a DLQ to another queue creates new messages {% cite awsDocsSqsDlqRedrive %}, so the age starts at 0 and  _ApproximateAgeOfOldestMessage_ can be trusted for redriven messages.)

<!-- TODO: should this be before quirks?-->
Despite the quirks, this metric is very useful: 
- If a message becomes too old (i.e. its age exceeds the queue's message retention period) then the message will be deleted {% cite awsDocsSqsWelcome %}.
  These messages don't get moved to a DLQ: they disappear.
  We can use _ApproximateAgeOfOldestMessage_ to warn us before this happens {% cite serverlessDevelopmentOnAws --locator 353 %}.
- _ApproximateAgeOfOldestMessage_ lets us track if the queue's consumers are keeping up with the queue's producers,
  and if we're meeting any target we may have for maximum time taken to process a message from the queue.

So what should we do with it?

<!-- TODO: should this be a table instead? -->
#### Alerting

For non-DLQs:
- Assuming your system should process every message, you should alert when the _ApproximateAgeOfOldestMessage_ approaches
  the retention timeout, with enough time for you to fix the issue {% cite serverlessDevelopmentOnAws --locator 353 %}.
  Estimate an upper bound for the time it will take you to solve the messages not being processed quickly enough
  and alert when _ApproximateAgeOfOldestMessage >= retention timeout - time required to solve the issue_.

  You might have a stricter target for the maximum time required to process messages on the queue - in which case, use that as the
  alert threshold instead.

  Depending on how sensitive your system is to losing messages (and how likely your queue is to receive poison-pill messages) you might
  want to make sure the queue's `maxReceiveCount` is at most 3 so messages don't get excluded from _ApproximateAgeOfOldestMessage_ and
  expire without you knowing.
- For some use cases, messages might only be worth processing within a strict timeframe.
  If that's the case, then you might choose to not have this alert at all.

For DLQs:
- Assuming your system should process every message, you should set a similar alert as for a non-DLQ, but use
  _ApproximateAgeOfOldestMessage >= retention timeout - time required to solve the issue - maximum time a message could have been on the original queue_
  (unless you have a stricter target to use instead).

  If your system really can't afford to lose a message, then _maximum time a message could have been on the original queue_ will
  have to be the original queue's retention period, and you will have to set a lower retention timeout on the non-DLQ than on the DLQ
  to make sure you have time to deal with messages that reach the DLQ before they disappear from it- which is the best practice that AWS recommend {% cite awsDocsSqsDlqRetention %}.



<!-- TODO: maybe add a summary? -->
## References

<!-- TODO: references in same year by same author don't have a,b, etc. so you can't tell them apart-->
{% bibliography --cited %}
