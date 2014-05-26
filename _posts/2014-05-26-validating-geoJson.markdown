---
layout: post
title:  "Validating GeoJson from the command line"
date:   2014-05-26 13:22:09
categories: geo cli nodejs
comments: true
---

This week I add a CLI to [geojson validation module](https://github.com/craveprogramminginc/GeoJSON-Validation) and will be going over how to use it in this post.  To start, install it `npm install geojson-validation -g`.

Next you need some geoJSON. We will use [this](https://raw.githubusercontent.com/wanderer/Detroit-Farm-Map/master/data/map.geojson) for the example. Now to check if it is valid or not simple run `curl https://raw.githubusercontent.com/wanderer/Detroit-Farm-Map/master/data/map.geojson | gjv`. If it was invalid `gjv` would print out the errors else it will just tell you its valid.

You can also validate local files like this `gjv file1 files2 ...`
