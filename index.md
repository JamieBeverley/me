---
layout: home
---

I am a full-stack software developer with a background in Human-Computer Interaction and 4 years of industry experience designing and deploying Machine Learning applications.

I work at Unity Health Toronto as the Director of Product Development for the Data Science and Advanced Analytics team (DSAA). At DSAA we develop machine learning and data science products that aim to improve patient outcomes and system efficiency at our 3 hospitals. Some examples of solutions we've developed and deployed include:
- **The St. Michael's Hospital Operations Centre**: a suite of data dashboards, alerts, and analytic tools aimed at improving patient flow.
- **CHARTWatch**: an early warning system for identifying patients at high risk of death.
- **The SMH Emergency Wait Times** dashboard: forecasts wait times in the ED.
- **The ED RN Assignment Tool**: an optimization tool and application for generating nurse scheduling assignments

...and many more things you can read about [here](https://lks-chart.github.io/blog/)...

Occasionally I write about technical topics [here](projects).

I am passionate about electronic music, modular synthesizers, and enjoy creating new experimental software for music performance (and sometimes I also get around to actually performing with it...). You can read more about that stuff [here](music). 

Please feel welcome to get in touch via [LinkedIn](https://www.linkedin.com/in/jamie-beverley-2a221695/).


<br/>

<h3>Recent posts</h3>
<ul>
  {% assign sorted = site.ramblings | sort: 'order' | reverse  %}
  {% for ramble in sorted limit:3 %}
    <li>
      <a href="{{  ramble.url | relative_url  }}">{{ ramble.title }}</a>
    </li>
  {% endfor %}
</ul>