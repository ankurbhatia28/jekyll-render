---
layout: post
title:  SEO Related Docs Experiments
date:   2021-04-19 10:00:00 -0800
categories: memo, seo
---

# SEO Related Docs

## One Sentence Description
This was a project set to increase SEO traffic by adding internal linking to our public published doc pages, served on mobile only.

## Background
Up until mid last year, Coda hadn't really explored many potential avenues of SEO growth earnestly. As such, the potential impact of many SEO techniques weren't completely understood. Personally, I had worked on a similar project at Airbnb, but more in a mentorship role for a junior DS to learn the technique we ended up using.

As a part of our growth strategy, we wanted to start increasing the amount of traffic we saw stemming from search traffic. Coda has quite a bit of first party content on the site. There is a gallery at the top nav bar of the homepage and we know that our published docs help folks discover use cases and inspire makers to start creating docs on Coda.



## The Goal
The avenue we wanted to explore was to add internal linking to our first and second party published docs. This is very common in many sites. At my previous company, Airbnb, along with optimizing keywords and title tags, this was one of the largest lifts we saw to our SEO traffic.

At Coda, we wanted to try by adding related links to the footer of published docs on mobile browsing.

While we had an idea that it would only benefit us to do so, we wanted to measure the potential impact it had on incoming mobile search traffic. As such, we wanted to set up an experiment to measure.

## Measurement Setup

The initial idea for the experiment was pretty straightforward. At the bottom of all published docs, add in a footer that contained related pages that users could click on and that search engines would recognize as internal links.

[This is the design we went with for the footer.](https://i.imgur.com/qLaHbNq.png)

We knew it would take some time for search engines (mainly Bing and Google) to completely recrawl all the pages and recognize the new structure, so it wasn't something we'd be able to measure effectively from day one. That was the first hurdle.

That could be solved with time. The slow exposure isn't ideal for an A/B experiment setup, but it was manageable and we could make the assumption that it affected control and treatment the same.

However, in discussing that, we naturally came across the second, and more blocking issue: there was no way for us to dictate a control and treatment for search traffic. Even if we set it up so that we would dynamically show the footer to specific visitors, we wouldn't be able to measure an increase in new search traffic by control and treatment, since that is a result of the aforementioned search crawling.

So we quickly abandoned the idea of a traditional A/B experiment. What we decided was to divide the corpus of our gallery published docs into two buckets (one treatment and one control) and only have the docs in the treatment contain the footer. We could then measure the search traffic stemming from each bucket to understand an impact.

This led to a difference-in-difference analysis. The idea behind this is both straightforward and clever. A difference-in-differences framework is a method that utilizes pre-experiment data to control for these baseline differences in the absence of any major changes. Essentially, we create a regression for the control and treatment in the pre-treatment phase and look at the changes in each post-experiment.
So we set up an estimator from a linear model, where for each page i and day t:

![Difference in Difference equation](https://i.imgur.com/8CiWZQR.png)

In this model, the principle variables are:

* _traffic<sub>it</sub>_ = number of landing page impressions to page _i_ on day _t_. There is a log transformation applied to account for its right-skew, since all published docs are not created equally. Also it helps regularize heteroskedasticity that is common in search traffic data.
* _treatment<sub>i</sub>_ = binary (1/0) on if the published doc is in the treatment group
* _post<sub>t</sub>_ = binary (1/0) on if the time is in the pre experiment or post experiment start period

Similarly, the mode allows for estimators to account for expected variations:

* _a<sub>i</sub>_ = fixed effect (mean) for a published doc
* _t_ = time index
* _dow<sub>i</sub>_ = weekday indicators

All this is more easily summed up visually:

![Difference in Difference visual](https://i.imgur.com/dsXuMNN.png)
_sourced from Airbnb blog post_

Where _b<sub>2</sub>_ is the post experiment difference in treatment and the expected treatment (counterfactual)


## Pre Launch Decisions

Not all traffic is created equally. The main decision to make before we launched the experiment was to decide how to break our corpus of published docs into control and treatment so that we may create a reasonable counterfactual. We wanted the traffic split to be as close to 50/50 as we could. However, while we knew of many popular published docs, we wouldn't be able to predict with enough accuracy which docs would take off. Often times, traffic is generated when someone with a large Twitter following links to a doc, or other scenarios out of our hand. Also, we wouldn't know how new docs would perform.

So we made the decision to exclude the top ~10% of docs. The upside is that it was much easier to break out the remaining hundreds of docs into cleaner, equally sized, and relatively stable groups. The downside is that we were excluding our largest traffic docs from the experiment, so we were likely under-estimating the impact the internal linking would generate. I messaged out to the team that this experiment would give us a lower bound of what to expect, which is the better situation to be in pre launch, I believe.

That was that, and **we launched** in June and I had my R script ready to take in data after a month and analyze the delta, which we had estimated would take until late September


## SNAFU

On a Monday morning in mid September, our site went down. :/

Usually, when this happens, there is an immediate war room with the engineers on call. As a data scientist, I'm not generally involved in those conversations. However, it was quickly identified that our experiment was a likely cause of the site going down. Given that we hadn't changed any fundamental underlying behavior of the product, I was confused. Also, we had shipped this in June -- what changed?

What had happened was that we were getting slammed by Bing's web crawler:

[This is a cleansed look at the change in traffic from known web crawlers.](https://i.imgur.com/oQFvjSy.png)


### The Postmortem

The traffic had 7x-ed nearly instantly. And the way that the we were showing the related docs in the footer was by running an expensive Postgres query each time someone had loaded a page. (Note: the query was expensive since it had to check the ACLs for every doc since we could only put discoverable docs in the footer). The crawler hit all our pages in a very short amount of time, overloaded our first of two PG servers, crashed it, shifted all the queries to the second server, subsequently crashed that one, and in doing so, took down our site.

Once the crawl traffic went back down, our memory consumption dropped, and the system coalesced.


### What I Learned
The positive was that we found a new failure mode for our website. That's really a glass half full look at it. We were not prepared to handle the increased load and I worked with the eng team to forecast how many more servers we would need to handle this sort of increase if it were to happen again (answer: 1 more server).

The site was down for roughly 8 minutes while the replica databases restarted and we added the third database less than 10 minutes afterwards.

We also worked to optimize the previously expensive queries by adding a layer of caching and doing some of the ACL check work offline using Airflow. I set up a table to do this that we incorporated into the experiment backend.


## The Analysis

Back to the fun stuff. The analysis was made possible by a package aptly called `CausalInference` developed by Kay Broderson.
Here is the output:

![Causual Inference Output](https://i.imgur.com/n6lO5yZ.png)

The relative effect of the treatment was ~6% with a 95% CI of `[2.5%, 10.5%]` with a p-value of `0.003`
We felt pretty good about shipping this and after it took the site down, it was as good a time as any to make a decision.


## The Conclusion

We had the idea that this would be good for our traffic but we didn't know by how much. This experiment gave us a lower bound and the confidence that these simple changes we could make for SEO were worth investing time and resources. The footers were the start of a new focus of our growth initiatives at Coda, focused solely on SEO. Today, we've made great improvements in keyword targeting, refining our title and meta tags, and making vast improvements to our core web vitals. And, as always, we continue to create meaningful content that resonates with our users.

I also learned that if you're not careful, you can inadvertently bring a website down. It is worth thinking through the potential impacts that site-wide changes may have on your underlying web frameworks and to optimize production queries whenever possible.
