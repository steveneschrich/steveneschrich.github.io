---
layout: default
title: "Creating a new ggplot type"
categories: [R]
author: steveneschrich
---

# Overview
I'm working on a paper with a collaborator and I'm running into a problem. ggplot is great, particularly the violin plots, but it doesn't quite do what I need. Specifically, I need the violins to be colored by density from green (low density) to yellow (high density). I found ggridges, which is close to what I need, except that the plots are one-sided (not a violin) and they overlap each other. So this prompted the idea of creating a new ggplot type that implements density-coloring for violin plots. I've never done it before, so I thought I'd document my process here.

# ggridges
