TODO:

+ make code fit into slides
+ make sure it's a story said by the code
+ extraction slides
  +  architecture
    + maybe just use UML component diagram?
    + create another image (REST vs GQL)
    + improve readability
- NOPE, no data image with stats per optimization attempt, just to show how it worked
+ image with monitoring solution


+ add images with stats
+ create a repated summary slide with `monitor-optimize-deploy`

* image with stats about first, failed attempt: failed cat jump or sth like this
~ last slides or two with the final thoughts
  ~ _fail earlier so you can succeed sooner_ ?
+ add summaries & quotes
  + flood of requests  memcached guy
  + power of joins - some database dude
  + server-side filtering -> maybe just the stats?, dog
  + use the domain  - Yoda
  + underfetching -
  + I don't always test on prod, but when I do I do it on Friday - prod testing guy
+ title, intro & about
+ images for talking
+ add funny images

+ which week was the one with Friday deploy?
  * 16:03 Aug 7th https://github.com/toptal/platform/pull/42515 -> first optimization
  * 15:43 Aug 28th https://github.com/toptal/platform/pull/41912 -> last optimization
* make sure you have time to tell about your feelings

Visual
* ruby sytax highlighing:
  * consts
  * ivars


Intro
+ focus on monitoring
+ symptoms of failures (e.g 429 errors)

Story
* show the cycle of monitoring (charts) - optimizing (code) - deploy and again monitoring (charts)

# Speaker notes

## Production failure
  * the hottest day of 2020
  * Tuesday, 28 Jul

## Summary
  * old techniques in new setting
  * threefold
  * learning
    * you need production traffic to see performance issues
  * simplicity
    * boring REST
    * generic minimal-surface API
  * tradeoffs
    * jitter
  * I told you about mistakes we made. Now you don't have to repeat our mistakes. Istead, go back to your work, and make your own mistakes. And if you want to know if you can safely make them I propose you a quick sanity check. Just think if you can deploy on Friday
