---
layout: post
title: "Meal Plan Journey"
date: 2024-01-28 20:10:36 -0800
categories: homelab web
---

Started by working on a proof of concept. Get a simple form where you can add a url from traders joes, and get the ingredients, picture and title of the recipe. Then see if I could run this in my home lab.

For the client, I used react and relay. The form uses react state to go through the steps, then I use a graphql mutation to pass the urls to the server and show loading indicators.

In the client the first challenge was to setup Relay. Bun is relatively new and there isn’t recommended code post processor that can replace the graphql query to relay code gen. So created a small bundling script and plugin that can do this transform.

In the server, I run Bun for http handling and create a chrome instance that is managed by puppeteer. This was smooth after figuring out how to use puppeteer’s text matching.

After having the prototype code complete I was able to run it locally; it was slow, but usable. Now the final feasibility test; make it run in my homelab that runs on 3 raspberry pi’s which are way more constraint than my m2 mini. To make it run there I created a docker container that bundles alpine, chrome and my bun server. Setted kubernetes and created a private docker repository in my homelab.

The kubernetes config was a bit esoteric but after ta bunch of copy pasting and trial and error I got it running. The main ramp up was on terminology, to got something running you need to create a pod that runs the app, this makes it run but not accessible to outside the kubernetes network. For this you need to setup an ingress, that describe the host names and routes to Services. Services map requests to pods. Wish this was a single config!

<pre class="mermaid">
        graph TD
        A[Client] --> B[Load Balancer]
        B --> C[Server1]
        B --> D[Server2]
</pre>

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

In the infra side, I needed persistent storage for the staging table and recipes. So installed mariadb in one rpi node. Initially I though about also having it as a kubernetes pod but couldn’t find a good win from doing so.

For 1-2 I just refactored my existing crawler and used bun as a script runner and the MySQL driver to use Mariadb. For setting up tables just sql scripts and the MySQL CLI. There are just 500 recipes in the gallery so in a single run I throttle to 4 page requests per 30 secs to be gentle and not get banned.

This got the recipes table available, now on to search. Iterating on this seemed easier to do in a Jupyter notebook. Cleaning the tables and playing around with different features to rank the results from. My first approach was to just use the title, but this wasn’t as effective, few results. I noticed I tend to search for ingredients more than titles so added them to the index and work better. Last step was to do some basic autocomplete to augment the results which ended up being good enough.

I didn’t want to commit cell evaluation so found a nice way to strip them through git pre processing. But required me to downgrade the notebook version.

After that I just copy pasted the code from the notebook into a single file with a simple http server and deployed it to the lab. Surprisingly this is already under a second so I just exposed it in the graphql schema in the bun web server.

From 3-5 I created a docker container with the pip requirements and add it to the private repository . It was surprisingly simple.

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
Opportunities

Misc:
vercel v0

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>
