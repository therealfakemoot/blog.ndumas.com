---
title: "Genesis Roadmap"
date: 2013-06-03T00:00:00Z

tags: ["genesis","python"]
series: ''
draft: false
---


Recently, I was working on an idea for a MUD; the general idea was to build a game with detailed 'inhabited' areas such as cities and dungeons as well as expansive wilderness regions for exploring and claiming by players. Naturally, I realized that doing this by hand would be unbearably tedious. A semi-realistic MUD world could contain 4,000,000 rooms; manually creating "Meadowy Plains #1421" would be error-prone and would drain the creative ability of a human. Thus, [Genesis](https://github.com/therealfakemoot/genesis-retired) was conceived. Note: this repoistory is now retired; the Python version of this project will not be pursued any longer.

# Moving Beyond the MUD
Planning a world generator for a MUD was an excellent idea, but very limited in scope. In particular, I realized how little I wanted to indelibly restrict myself to [Evennia](http://evennia.com/) as the framework in which the creation process happened. Evennia offers a great deal of power and flexibility, but the restriction of MUD concepts got me thinking that I should generalise the project even further.

In the end, I want Genesis to be a completely standalone world-design toolkit. The target audience includes tabletop gamers who need a physical setting for their campaigns, as well as authors who can create characters and stories, but have trouble with the tedium of drawing coastlines and mountain ranges out by hand.

# The Vision

As of the time of writing, implementation of Phase 1 is only partially completed. What follows is my overall goals for what each phase can accomplish.

## Phase 1: Heightmap Generation
Using a [simplex](http://en.wikipedia.org/wiki/Simplex) function, I populate an array with values representing the height of the world's terrain at a given coordinate. This is pretty simple; it's a pure function and when run with PyPy it absolutely screams. There's little to be said about this step because it produces the least interesting output. If so desired, however, a user could take the topological map generated and do whatever they please without following any further phases.

## Phase 2: Water Placement
The water placement phase is the simplest phase, yet has the most potentially drastic consequences in further phases. Given a heightmap from Phase 1, the user will be able to select a sea-level and at that point all areas of the map with a height below sea level will be considered underwater. This step can be reapplied to smaller subsections of the map allowing for the creation of mountain lakes and other bodies of water which are not at sea-level.

## Phase 3: Biome Assignment
Biome assignment is a rather complex problem. Full weather simulations are way beyond what one needs for an interesting map. To this end, I've found what I believe to be two excellent candidates for biome classification systems.

{{< figure src="/img/two_biome.jpg" caption="Two-axis Biome Chart">}}

This graph uses two axes to describe biomes: rainfall and temperature. This is an exceedingly easy metric to use. Proximity to water is a simple way to determine a region's average rainfall. Temperature is also an easy calculation, given a planet's axial tilt and the latitude and longitude of a location.

{{< figure src="/img/tri_biome.png" caption="Three-axis Biome Chart">}}

This graph is slightly more detailed, factoring in a location's elevation into the determination of its biome. As this phase is still unimplemented, options remain open.

## Phase 4: Feature Generation
In the Milestones and Issues, I use the term 'feature' as a kind of catch-all; this phase is the most complex because it involves procedural generation of landmarks, cities, dungeons, villages, huts, and other details that make a piece of land anything more than an uninteresting piece of dirt. This phase will have the most direct interaction by a user, primarily in the form of reviewing generated features and approving or rejecting them for inclusion in the world during Phase 5. In this Phase, the user will determine what types of features they desire (large above ground stone structures, small villages, underground dungeons, and so on).

## Phase 5: Feature Placement
Phase 5 takes the objects generated during Phase 4 and allows the user the option of manually placing features, allowing Genesis to determine on its own where to place them, or some combination of both. Although it wouldn't make much sense to have multiple identical cities in the same world, this phase will allow duplication of features allowing for easy placement of templates which can be customised at some future point.

# In Practice
The Genesis Github repository currently has a working demo of Phase 1. CPython is exceedingly slow at generating large ranges of simplex values and as such, the demo will crash or stall when given exceedingly large inputs. This is currently being worked on as per #8.
