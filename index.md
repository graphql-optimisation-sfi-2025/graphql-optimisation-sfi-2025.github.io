# Monitor, Optimize and Deploy

### (on Friday)

::::

Friday afternoon, deploy or not to deploy?

::::

Intro

::::

Safe environment
* engine
* direct calls & HTTP calls
* feature flag, percentages
* fallback
* monitoring
  * errors
  * performance (response time)
  * source (stacktrace)
  * params

::::

First attempt
* replace AR associations with REST calls
* deduplicate & optimize REST calls
* epic failure (give the numbers)


Another attempt
* GQL
* step by step migration
* another layer of feature flags

In general OK, but some notable issues, mostly on the consumer side

::::

Flood of requests

* Problem: single view/job initiates many billing requests
  * how many? Thousands!!!!
* solution: preload & cache
> First solution: load all and cache
::::

Flood of DB queries
* Problem: ever element of a collection of billing records needed platform data (product)
* Solution: client-side join (hash join)

> Hash joins to the rescue
::::

Too much data

* Problem: we had generic queries that fetched all the data that is needed
* Solution: query customization (GQL shines here), underfetching, moving the filtering to the server side
> ?

::::

Side story: I broke production, because I didn't test all the params and by default we were returning _all_ the data.

  **how many records were we trying to return?**
> You can break stuff sometimes, if you can fix it quickly

::::

* Problem: frequently used field
* First solution: build a read model based on kafka events
* Second solution: use local data!
> Use the domain, Luke!

::::

All simple solutions applied, we still have
