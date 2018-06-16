---
permalink: /:categories/:title/
layout: post
title: "Using RStudio on a Google Cloud Dataproc cluster"
date: 2018-05-16 12:00:00.000000000 +00:00
type: post
published: true
status: publish
categories:
- blog
image: https://owenjonesuob.github.io/assets/featured/dataproc.jpg
tags:
- r
- rstudio
- cluster
- google
- cloud
- dataproc
- sparklyr
excerpt: Getting set up to use Spark with R on a Google Cloud computing cluster.
---

A few months ago, I was working on a really cool project with Google.

Without going into too much detail, we were extracting structured information from unstructured text and image data, and then using that new structured information to build a recommender system and some other fun ML stuff.

Anyway, a fairly major aspect of this project was the availability of [Google Cloud Platform](https://cloud.google.com) resources. Given that I was expecting to do a lot of heavy image processing, I thought it would be useful to get a Dataproc cluster set up for a bit of extra _oomph_.

Dataproc uses Spark to manage the underlying cluster, and by default the cluster machines are set up with Python (and PySpark). However, I had already done a bunch of work in R on my own computer, and I was hoping to get going quickly on the cloud cluster without having to translate everything into Python first. Therefore I resolutely decided to get RStudio and sparklyr up and running on my shiny new Dataproc cluster.

Unfortunately... it seemed like nobody had actually done this before!

After a lot of digging, bodging and cussing, I did eventually succeed - and before I was able to feel too bad about spending so long on it (longer than it would have taken to rewrite my work in Python!), our colleague at Google said something along the lines of:

> "Hey, this is pretty cool. Do you think we could write this up properly?"

Sure, why not! I put together some brief comments and dodgy screenshots; shortly afterwards the project finished, and I promptly forgot any of this had ever happened.

Then a couple of days ago I received an email. The wonderful people at Google have worked their magic and turned my just-functional setup process into a full Cloud Solutions writeup! I am indebted to them for doing so, and you can read it here:

> [Running Rstudio Server on a Cloud Dataproc Cluster](https://cloud.google.com/solutions/running-rstudio-server-on-a-cloud-dataproc-cluster)