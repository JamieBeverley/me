---
layout: page
title: 'Yet Another Model Server' 
order: 2
---

- A repeating set of needs we see in our data science projects when we go to deploy models:
  - A (usually HTTP) service/API to serve predictions
  - a database that saves predictions and other relevant contextual info relating to each prediction (e.g version model, perhaps some features)
  - An easy mechanism to version and deploy models
  - An internal dashboard and alerts for monitoring model (and feature?) drift
- Ideally we wouldn't prescribe any limitations on what tools/frameworks data scientists use
- I'm prototyping a service which includes:
  - An API gateway for proxying requests to model HTTP services
  - Middleware which saves predictions to a database
  - A language-agnostic API for publishing new models which run as containers
  - Some minimal orchestration for spinning up/spinning down model services based on demand (likely later using something like K8s or Nomad instead of a loose mess of containers)
- Some preliminary progress in python (Django) which I hope to rewrite in Go.
