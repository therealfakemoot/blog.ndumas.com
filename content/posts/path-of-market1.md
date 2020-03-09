---
title: "Path of Market: Part 1"
date: 2019-07-08T10:45:07-04:00
draft: false
toc: true
images:
tags:
  - golang
  - prometheus
---

Path of Exile is an ARPG similar to Diablo: procedurally generated maps, kill monsters to get loot so you can kill monsters faster. It's pretty fun and offers a really flexible build system that allows for a lot of creativity in how you achieve your goals. Of particular interest is the API exposed by the development team.

# Stashes

Each character has a set of "stashes". These are storage boxes which can be flagged a public. Public boxes are exposed via the [api](https://www.pathofexile.com/developer/docs/api-resource-public-stash-tabs) endpoint. This API is interesting in how it handles paging; each request gives you an arbitrary number of stashes, and a GUID indicating the last stash provided by the API. Subsequent requests to the API can include the aforementioned GUID in the `id` url parameter to request the next batch of stashes. This is a sorta crude stream, in practice. Maybe one day they'll reimplement it as a websocket API or something fun like that.

# The Market

This API is what powers the market for the Path of Exile community. There's [quite](https://poe.watch/prices?league=Legion) [a](https://poe.trade/) few sites and tools leveraging this API, including the [official market site](https://www.pathofexile.com/trade/search/Legion). These market sites are very handy because they offer complex search functionality and various levels of "live" alerting when new items become available.

What I found fascinating, though, is the ability to monitor trends, more than finding individual items. As a former EVE player, I was used to relatively advanced market features like price histories, buy/sell orders, advanced graphing options etc and it's something I've missed everywhere I've gone since. After some investigation, I found that Prometheus and Grafana could offer a powerful base to build upon. Prometheus is a tool for storing time-based "metrics", and Grafana is a visualizer that can connect to Prometheus and other data sources and provide graphs, charts, tables, and all sorts of tools for seeing your data. Below is an example of a chart showing memory usage on a Kubernetes pod.

{{< figure src="/img/grafana-mem-usage.png" caption="Grafana memory usage chart">}}

# First Steps

Obviously, the first step is talking to the official Path of Exile API and getting these stashes into a format that I can work with programmatically. The JSON payload was moderately complex, but with the help of some [tooling](https://mholt.github.io/json-to-go/) and unit testing I was able to build out some Go structs that contained all the metadata available.

A particularly fun challenge was this one, describing how "gem" items could be slotted into an item. This was a challenge because the API can return either a string *or* a boolean for a specific set of fields. This is, in my opinion, not as "well behaved" API but you don't always get the luxury of working with ones that are well behaved. This unmarshaling solution helps account for this inconsistency and populates the relevant fields accordingly.

{{< highlight go "linenos=table" >}}

type SocketAttr struct {
        Type  string
        Abyss bool
}

func (sa *SocketAttr) UnmarshalJSON(data []byte) error {
        var val interface{}

        err := json.Unmarshal(data, &val)
        if err != nil {
                return err
        }

        switch val.(type) {
        case string:
                sa.Type = val.(string)
        case bool:
                sa.Abyss = val.(bool)
        }
        return nil
}

type SocketColour struct {
        Colour string
        Abyss  bool
}

func (sc *SocketColour) UnmarshalJSON(data []byte) error {
        var val interface{}

        err := json.Unmarshal(data, &val)
        if err != nil {
                return err
        }

        switch val.(type) {
        case string:
                sc.Colour = val.(string)
        case bool:
                sc.Abyss = val.(bool)
        }
        return nil
}

{{< /highlight >}}

With that done, I had passing tests that parsed a variety of sample items I had manually extracted from the API. Next was turning these into a "stream" that I could process. Channels seemed like a natural fit for this task; the API did not guarantee any number of results at any time and only declares that you periodically request the last ID you were given by the API.

The full code is [here](https://github.com/therealfakemoot/pom/blob/master/poe/client.go), but I'll highlight the parts that are interesting, and not standard issue HTTP client fare.

{{< highlight go >}}
		d := json.NewDecoder(resp.Body)
		err = d.Decode(&e)
		if err != nil {
			sa.Err <- StreamError{
				PageID: sa.NextID,
				Err:    err,
			}
			log.Printf("error decoding envelope: %s", err)
			continue
		}
		log.Printf("next page ID: %s", e.NextChangeID)
		sa.NextID = e.NextChangeID

		for _, stash := range e.Stashes {
			sa.Stashes <- stash
		}
{{< /highlight >}}

This snippet is where the magic happens. JSON gets decoded, errors are pushed into a channel for processing. Finally, stashes are pushed into a channel to be consumed outside inside the main loop. And here's where I'll leave off for now. There's quite a bit more code to cover, and I'm still refactoring pieces of it relatively frequently, so I don't want to write too much about things that I expect to change.
