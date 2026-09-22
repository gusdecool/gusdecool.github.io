# Benchmark Jev AI from TypeSafe

I tried Jev AI from [TypeSafe](https://typesafe.ai/) to see if it really faster and can reduce the AI cost.

Source code available at [https://github.com/gusdecool/experiment/blob/main/jev/bench.ts](https://github.com/gusdecool/experiment/blob/main/jev/bench.ts)

## What is Jev

Jev is an AI model from TypeSafe, that can answer question (query) and return type safed result,
with the aim to make it faster and more cost-effective than other AI models.

Imagine when you develop application and need a classification model so you can setup "if/else" or "switch case" statement.
Jev can help you to do that.

Using Jev feels like when we built probabilistic model with machine learning approach but didn't need to train the model.

## How I run the test

Compared it with Gemini flash latest (cheap model). At the time of writting, it at version 3.8.

Asked classification question "I was charged twice for my subscription this month. Please refund the extra charge."
Then AI classifies the ticket into `[billing, technical, other]`.

Then have Gemini simulate the same situation to compare the speed & cost.
I ran the same questions 5 times to get average result.

## Benchmark result

The AI tested from **APAC** area, so it may also affect latency.

Full result

```shell
bun run jev/bench.ts
Classifying ticket into [billing, technical, other], 5 runs per provider.

Jev (jev-latest):
  [jev] run 1/5: 1034ms -> "billing" (jev-1.13.0)
  [jev] run 2/5: 377ms -> "billing" (jev-1.13.0)
  [jev] run 3/5: 354ms -> "billing" (jev-1.13.0)
  [jev] run 4/5: 323ms -> "billing" (jev-1.13.0)
  [jev] run 5/5: 394ms -> "billing" (jev-1.13.0)

Gemini (gemini-flash-latest):
  [gemini] run 1/5: 1995ms -> "billing" (gemini-3.8-flash)
  [gemini] run 2/5: 1749ms -> "billing" (gemini-3.8-flash)
  [gemini] run 3/5: 1272ms -> "billing" (gemini-3.8-flash)
  [gemini] run 4/5: 1742ms -> "billing" (gemini-3.8-flash)
  [gemini] run 5/5: 2878ms -> "billing" (gemini-3.8-flash)

=== Results ===

Jev (jev-latest -> jev-1.13.0):
  latency  min=323ms  avg=496ms  p50=377ms  max=1034ms
  tokens   avg_input=316.0  avg_output=38.0
  cost     ~$0.000013 per call (launch pricing)

Gemini (gemini-flash-latest -> gemini-3.8-flash):
  latency  min=1272ms  avg=1927ms  p50=1749ms  max=2878ms
  tokens   avg_input=49.0  avg_output=1.0
  cost     ~$0.000040 per call (standard paid tier pricing)
```

### Jev result

```shell
Jev (jev-latest):
  [jev] run 1/5: 1034ms -> "billing" (jev-1.13.0)
  [jev] run 2/5: 377ms -> "billing" (jev-1.13.0)
  [jev] run 3/5: 354ms -> "billing" (jev-1.13.0)
  ....
```

Jev initially run slower at `1k ms`, but afterwards being so fast at `~300ms` which most likely due to API latency for round-trip.
Looks like Jev need time to initialize the question, saved it into cached and reused it.

### Gemini Flash

```shell
Gemini (gemini-flash-latest):
  [gemini] run 1/5: 1995ms -> "billing" (gemini-3.8-flash)
  [gemini] run 2/5: 1749ms -> "billing" (gemini-3.8-flash)
  [gemini] run 3/5: 1272ms -> "billing" (gemini-3.8-flash)
```

Gemini is consistenly averaging at `~1700ms`. Perhaps Gemini didn't have cache functionality at all.

### Compared

```shell
Jev (jev-latest -> jev-1.13.0):
  latency  min=323ms  avg=496ms  p50=377ms  max=1034ms
  tokens   avg_input=316.0  avg_output=38.0
  cost     ~$0.000013 per call (launch pricing)

Gemini (gemini-flash-latest -> gemini-3.8-flash):
  latency  min=1272ms  avg=1927ms  p50=1749ms  max=2878ms
  tokens   avg_input=49.0  avg_output=1.0
  cost     ~$0.000040 per call (standard paid tier pricing)
```

Even though Jev's token usage looks higher, it's still cheaper than Gemini.
Jev's latency is also consistently much lower than Gemini.

## How I will use this

Considering Jev capable to do the task while being **5 times faster** & **3 times cheaper**.
It's ideal to integrated Jev into my AI workflow project where classification is needed.

Although in general, I would always setup the scenario where if Jev server down, it will fallback to other existing model.
