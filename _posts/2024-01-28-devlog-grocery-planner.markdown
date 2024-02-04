---
layout: post
title: "Building a grocery planner"
date: 2024-01-28 20:10:36 -0800
categories: devlog homelab
has_code: true
---

I wanted to build a grocery planner as a toy app for my homelab. The idea is to get a web app where I
can look up for recipe names or ingredients and get the list of groceries to buy. On the way,
I wanted to learn about how to setup and coordinate servers through kubernetes and try out [Bun](https://bun.sh/),
[Grats](https://github.com/captbaritone/grats) and [Relay](https://github.com/facebook/relay) for web development.

I'll go over my initial reactions about using these for the first time in a project and I iterated
over the design of the app. The sections to cover are:

- **Proof of concept**: First step to get the minimum version working, what I had to build to get it done and
  what is the initial design. App work's but is slow.
- **Job Queues**: Move from executing everythign in a single request to a working queue for scraping jobs. Recipe
  scraping works but search doesn't feel good.
- **Recipe Search**: Use an inverse index and basic heursitics, prototype using a jupyter notebook. Recipe search
  works and feels okay but is not integrated to the app.
- **Typeaheads**: Moving from prototying in a notebook to end 2 end working web app. It works okayish and is fast!

I'll try to approximate how much load this system might be able to handle. You can see the final code [here](https://github.com/qsiandre/meal-plan).

## Requirements

- Homelab: I have 3 raspberry pi's connected to the local network and with kubernetes running, but all can fit in 1[[1]](#1).
- Docker: Setup a local docker repository for kubernetes to access[[1]](#1).
- MariaDB: MySQL database to hold data beyond requests[[2]](#2).
- Jupyter and Conda: Fast prototyping environment to have ready in your development environment[[3]](#3).

## Proof of concept

Started by working on a proof of concept. Get a simple form where you can add a url from traders joes,
and get the ingredients, picture and title of the recipe. Then see if I could run this in my home lab.

I'm get design paralysis, so tried to get some inspiration form vercel.v0 and got an initial mock UI that
I can implement. I'm more familiar with the front end, so was easier for me to start there and get some
wins going.

For the web app, I wanted to use Bun as the web server, I read that it's fast and it's an all batteries included
runtime. Went with Relay to see how easy would it be to setup and get a template running for other projects based on Bun.
Relay requires a GraphQL server schema, and wanted to test Grats as it replicates the code first approach I'm used, in
the past getting GraphQL schemas was prone to error because it felt like a lot of duplicated work to define the types
both manually in the .graphql file and typescript.

In the client the first challenge was to setup Relay. Relay depends in a code transform to extract the GraphQL fragments so
it can generate the types again the GraphQL Schema. Bun is relatively new and there isn’t recommended post processor
that can replace the graphql string template literals with references to teh relay module. So created a small bundling script and
plugin that can do this transform. More details [here](https://github.com/qsiandre/meal-plan/blob/main/plugins/relayPlugin.ts).

```typescript
export const relay: BunPlugin = {
  name: "extract relay",
  setup(build) {
    build.onLoad({ filter: /\.(tsx|ts)$/, namespace: "file" }, ({ path }) => {
      const file = readFileSync(path, "utf8");
      const contents = compile(path, file, {
        artifactDirectory: "__generated__",
      });
      // this was the trickiest part, didn't now that you could pass the content
      // to the tsx loader...
      return { contents, loader: "tsx" };
    });
  },
};
```

In the server, I run Bun as web server, there is an example of Grats using yoga that runned out of the box after connecting it to a Bun request handler.
Creating a chrome instance that is managed by puppeteer was really smooth, I had some Bun scripts running for quick prototyping. I find using Bun pretty
easy, most commands are similar to what you node has, and some common usecases are built in, like reading environment variables.

The form uses React state to go through the steps, when all URLs are ready there is a single GraphQL mutation that
kicks off the scraping in the server. Relay then helps with the loading states and type generation, finally when the
mutation finishes, it returns a list of recipes that are stored in React state and passed around components.

I decided to go with this approach because:

1. Starting Chrome and navigating to the URL, loading JS and scraping the HTML in single request will take 10+ secs. So `useLazyLoadQuery`
   wasn't as ergonomic, as I would need to change the params and rely on Suspense for this. `useMutation` in the other hand exposes both the
   request and a bool that you can use to imperatively show a loading indicator.
2. Reading directly from Relay without React state seemed like an overkill, most of the interactions in the stepper don't require server
   data so the benefits of colocation don't apply.
3. The biggest unknown was on search, and setting up jobs for scraping. If I need to colocate more server reads I can just refactor and Relay
   makes it really easy to iterate on.

After having the prototype code complete I was able to run it locally; it was slow, but usable. Now the final
feasibility test; make it run in my homelab that runs on 3 raspberry pi’s which are way more constraint than my m2 mini.
To make it run there I created a docker container that bundles alpine, chrome and my bun server.
Setted kubernetes and created a private docker repository in my homelab.

The kubernetes config was a bit esoteric but after a bunch of copy pasting I got it running. The main ramp up was on terminology, after
trying and failing multiple times got used to it. So `Ingress > Service > Pod`.

- Pods are where you can run containers, you can see their logs whe they are deployed using `kubectl logs <pod-id>`, also `kubectl describe pod <pod-id>`
  will hint about errors.
- Services are the middle layer, used to expose pods to the kubernetes cluster. This can be debugged by port forwarding `kubectl port-forward <service-id> <to>:<from>`, if this fails
  to access your pod, there is something wrong with the pod. This will get important while setting up search.
- Ingress describe the externally accessible URLs, and will redirect the traffic to services. You test them just typing the URL in the browser. If it fails it's a DNS issue.
  For my small toy app, was an overkill but I can see how this can get usefull as you deploy something more complicated and want to add more pods to support your traffic. Wish this was
  in a single step for quick iteration.

<pre class="mermaid">
  graph LR
  Browser --> Relay --> Grats --> Chrome
</pre>

The hot path is scraping, it consumes 300MB of memory per request, takes 10 - 20 seconds to finish. Since I have 3 machines with 4GB of memory, it could create aprox 10 with
400MB request limit pods. I can run 4 queries each minute assuming each query takes 15 seconds. So QPS is `4 / 60 * 10 = 0.66 QPS`. To go beyond this need to move Chrome out
of the web request through jobs that scrape recipes and store them in a DB. since I'm mostly interested in Recipes from Trader's Joes and those don't change frequently this is an okay tradeoff.

## Job queues

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

## References

- <a id="1">[1]</a> Great tutorial, it goes deep into how to make it as close as prod https://rpi4cluster.com/
- <a id="2">[2]</a> The goat https://pimylifeup.com/raspberry-pi-mysql/
- <a id="3">[3]</a> Simplest setup https://medium.com/@nrk25693/how-to-add-your-conda-environment-to-your-jupyter-notebook-in-just-4-steps-abeab8b8d084

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true });
</script>
