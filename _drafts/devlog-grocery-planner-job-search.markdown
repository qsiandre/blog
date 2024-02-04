---
layout: post
title: "Building a grocery planner - search"
categories: devlog homelab
---

First feedback from my dear only user was that it was too slow and copying urls is also work. She would like to just write the recipe name or ingredients and get options.

So need to go from 10 secs to 100ms and make some recommendations.

The 10 seconds come from spinning up chrome and waiting for the page to load. Can’t use just http because I need to wait for the whole JS to load and render to get the content. Additionally to do recipe search I’ll need to have all content and recipes to ready! No time to wait for crawling!

To get this working I:

1. Created a gallery scraper that can give me urls daily and adds the to a staging table
2. Scrape worker, takes recipes from the staging table each 15 mins and scrapes the content
3. Create a service that gets the latest version of the recipes, creates an inverted index.
4. For the last word, get the top 3 words in our recipe vocabulary than share prefix and add them as alternative query.
5. Given the sentence and limit get all the entries that contain any word in the sentence and sort them by frequency.

The state machine for the crawler is this, each day get all the recipes. Get a single worker that gets the recipes and marks them as in progress if it fails, mark them back to available so they get picked next. I could speed up and scale through multiple workers by assigning a worker by parity.

This was interesting to think through, while it is over engineered now as I could just run a single script. It made it really easy to test, I can just deploy the k8s recipe and inspect the sql tables.

For 1-2 I just refactored my existing crawler and used bun as a script runner and the MySQL driver to use Mariadb. For setting up tables just sql scripts and the MySQL CLI. There are just 500 recipes in the gallery so in a single run I throttle to 4 page requests per 30 secs to be gentle and not get banned.

## Recipe Search

This got the recipes table available, now on to search. Iterating on this seemed easier to do in a Jupyter notebook. Cleaning the tables and playing around with different features to rank the results from. My first approach was to just use the title, but this wasn’t as effective, few results. I noticed I tend to search for ingredients more than titles so added them to the index and work better. Last step was to do some basic autocomplete to augment the results which ended up being good enough.

I didn’t want to commit cell evaluation so found a nice way to strip them through git pre processing. But required me to downgrade the notebook version.

After that I just copy pasted the code from the notebook into a single file with a simple http server and deployed it to the lab. Surprisingly this is already under a second so I just exposed it in the graphql schema in the bun web server.

From 3-5 I created a docker container with the pip requirements and add it to the private repository . It was surprisingly simple.

## Typeaheads

Finally, I needed to add the typeahead to the web app and replace the mutation to accept ids instead of URLs. Initially I tried to use a Relay connection defined in the client that’s updated each time a mutation finish but then I just got lazy and used react state and fetchQuery tied with a useEffext. This was good enough.

The original then replaced the mutation arguments to accept ids, and added a query to the db instead of call to puppeteer and everything worked!

This was good enough and the final design is.

<pre class="mermaid">
        graph TD
        A[Client] -->|tcp_123| B
        B(Load Balancer)
        B -->|tcp_456| C[Server1]
        B -->|tcp_456| D[Server2]
</pre>

Cost
Qps

## Opportunities

Misc: vercel v0

## Conclusion
