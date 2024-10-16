---
layout: home
---

I am a full-stack developer based in Toronto, Canada with a background in Human Computer Interaction, ML, New Media, and Critical Theory.

I'm as excited by building software as I am by critiquing its social value; a tension (at times dissonance) that I find endlessly fascinating and strangely productive.

I work for the City of Toronto's Technical Services Division, focusing on building and maintaining the backend of Toronto's [Open Data Portal](https://open.toronto.ca/), which provides public access to datasets pertaining to the City's operations.

Previously I worked at Unity Health Toronto as the Director of Product Development for the Data Science and Advanced Analytics team (DSAA). My team and I were responsible for the Application Engineering and Design of tools that leveraged machine learning and/or data science to improve patient outcomes and system efficiency at our hospital sites. Some examples of solutions we developed and deployed include:
- **The St. Michael's Hospital Operations Centre**: a suite of data dashboards, alerts, and analytic tools aimed at improving patient flow.
- [**CHARTWatch**](https://www.cbc.ca/news/health/ai-health-care-1.7322671): an early warning system for notifying care providers when a patient is at high risk of death or deterioration.
- **The SMH Emergency Wait Times** dashboard: forecasts wait times in the ED.
- **The ED RN Assignment Tool**: an optimization tool and application for generating daily nurse scheduling assignments.

...and many more things you can read about [here](https://lks-chart.github.io/blog/)...

I am passionate about electronic music, modular synthesizers, and enjoy creating new experimental software for music performance (and sometimes I also get around to actually performing with it...). Occasionally I share more about that stuff [here](music).

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
