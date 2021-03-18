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

:::

```ruby
# AR model
class Product
  def billing_records
    @billing_records ||= Billing::QueryService
      .billing_records_for_products(self)
  end
end
```
```ruby
# Business logic
Product.eligible.find_in_batches.each do |p|
  DoBusinessLogic.call(p) # one billing request per call
end
```

:::

```ruby
Product.eligible.find_in_batches do |products|
  cache_billing_records(products).each do |p| # one billing request per batch
    DoBusinessLogic.call(p) # no billing requests, billing records cached
  end
end

def cache_billing_records(products)
  # array of billing records
  billing_records = Billing::QueryService
    .billing_records_for_products(*products)

  indexed_records = billing_records.group_by(&:product_gid)

  products.each do |p|
    e.cache_billing_records!(indexed_records[p.gid].to_a) }
  end
end
```

::::

Flood of DB queries
* Problem: ever element of a collection of billing records needed platform data (product)
* Solution: client-side join (hash join)

> Hash joins to the rescue

:::

```ruby
# billing model in platform
class BillingRecords
  def product
    @product ||= Product.find(product_id)
  end
end
```

```ruby
# business logic
  billing_records = Billing::QueryService
    .billing_records_for_products(*products)

  billing_records.each do |r|
    BusinessLogic.call(r, r.product) # one query to products table per call
  end
```

```ruby
  billing_records = Billing::QueryService
    .billing_records_for_products(*products) # with product assigned to billing records

  billing_records.each do |r|
    BusinessLogic.call(r, r.product) # one query to products table per call
  end
```
:::

```ruby
def product_billing_records(products)
    products_by_gid = products.index_by(&:gid)
    product_gids = products_by_gid.keys.compact
    return [] if product_gids.blank?

    billing_records = fetch_billing_records(product_gids: product_gids)

    billing_records.each do |billing_record|
        billing_record.preload_product!(
            products_by_gid[billing_record.product_gid]
        )
    end
end
```

::::

Too much data

* Problem: we had generic queries that fetched all the data that is needed
* Solution: query customization (GQL shines here), underfetching, moving the filtering to the server side
> ?

Underfetching
1.
```
REST fetching 20+ fields and 2-3 associated collections
```
```
GQL with 4 fields
```
2. Generic GQL mimicking REST => GQL with 3-4 fields an no associations

Filtering:
```
def billing_records_for_products(*products)
  fetch_billing_records(filter: {products: products}).
    select(&:accessible?)
end
```
vs

```
def billing_records_for_products(*products)
  fetch_billing_records(filter: {products: products, accessible: true})
end
```

And I deployed and waited, and...

::::

Side story: I broke production, because I didn't test all the params and by default we were returning _all_ the data.
How I felt
How we reacted
What my manager did

  **how many records were we trying to return?**
> You can break stuff sometimes, if you can fix it quickly

:::

> something is wrong with sidekiq workers, they're consuming too much memory

> look, another master build failed to deploy to staging

> DM: Hey, your build seem to break staging deployment

> Hey, is platform having some issues right now?

> platform is down

:::

memory screenshots

:::

root cause:

client:
```
  get('/records', **params.slice(:product_gids))
```

query:
```
def billing_records(product_gids: nil, gids: nil, client_gid: nil)
  scope = Product
  scope = scope.where(product_gid: product_gids) if product_gids
  scope = scope.where(gid: gids) if gids
  scope = scope.where(client_gid: client_gid) if client_gid
  scope.all
end
```

fix:
```
def billing_records(product_gids: nil, gids: nil, client_gid: nil)
  return [] if [product_gids, gids, client_gif].all?(&:blank?)
  # ...
end
```

::::

* Problem: frequently used field
* First solution: build a read model based on kafka events
* Second solution: use local data!
> Use the domain, Luke!

```
::Billing::QueryService
  .first_successful_record_created_at(client)&.in_time_zone&.to_date
```
Plan
* add field to kafka
* build a read model
* backfill data
* start using the read model
* remove billing query

Solution
* find that date in local DB
* verify if it's really the same date
* use it and remove billing query

```
client.products.successful.minimum(:start_date) # one DB query
```

::::

* 429 on Sunday evening, every week, *couldn't replicate locally*
* All simple solutions applied, we still have 429 every Sunday evening...
* reminders sent to our talents (thousands of users)
* talents around the world, but some TZ more popolous than other, 25% of talents in one TZ
* every reminder in a separate sidekiq job, no preloading possible, hundreds of jobs try to load billing data at the same time, 5pm in the most popular TZ
* can't move talents to other TZs, so moving reminders a bit (+-2min) 120s timespan should be enough for all the requests (just a few requests/s)
* no effect!
* sidekiq polling, implemented rate limiting with `Sidekiq::Limiter` (enterprise feature)
* worked!

> Safe env on production let you fix unreproducible errors

:::

Problem:
```
eligible_products.each do |p|
  WeeklyReminder.schedule(product, day: :sunday, time: 17) # scheduling at talent's 5 PM on Sunday
end
```

1.
```
eligible_products.find_in_batches do |batch|
  with_billing_cycles_preloaded(batch) do
    batch.each do |product|
      WeeklyReminder.schedule(product, day: :sunday, time: 17) # scheduling at talent's 5 PM on Sunday
    end
  end
end
```

2 & 3
optimized GQL query +
```
def scheduling_time(*)
  super + (SecureRandom.rand * 120 - 60).seconds
end
```

4
Sidekiq::Limiter, window limiter (image)

and one of those (probably 3) was one of those when I deployed on Friday. I hope now you understand.

::::

Techniques that helped us:

Monitor:
* instrumentation
* logs, error tracker, new relic

Optimize
* preloading to avoid N+1
* app-level hash joins
* using local data
* underfetching
* spreading the load

Deploy
* safe env with a fallback
* feature flags with gradual enabling
* easy & reliable rollback

Nihil novi!

::::

Nihil novi!
Optimize
* preloading to avoid N+1 -> any ORM
* app-level hash joins -> even MySQL has hash joins now
* using local data instead of fetching it
* underfetching -> SELECT * vs SELECT field1, field2
* spreading the load -> known for years (read about it )

Why not good from the beginning?
* we started with a boring solution and then applied improvements
* easy to overlook perf degradation while refactoring
* hard to find perf issues by staring at the code
* DRY vs YAGNI - one, big universal endpoint vs several similar, smaller ones optimized for a job (btw GQL solves this)
* => we could rediscover all those things in the "Optimize" stage, because of the hidden work of the "Monitor" and "Deploy" stages

> We're sharing this so that you don't repeat our mistakes.
> Go, make your own mistakes, come back and share your story.
> That's how we learn as a community and as an industry: we inspect/monitor, we do/optimize and we deploy and then we share what we found out.
