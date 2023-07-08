---
layout: page
title: 'Developing Dashboards for Passive Displays'
order: 4
---

Developing an application that runs on a passive display in a shared space can
have additional complexity.

We needed to deploy an existing React/Redux data dashboard across 40 TVs of
varying resolution accross our organization.
A summary of some requirements:
- must update every minute
- may be displayed in some public areas, in which sensitive data
must be masked/obfuscated for privacy reasons
- must automatically launch when the PC connected to the TV turns
on
- we'd like to minimize additional overhead in maintaining the TVs (obviously). 
Deploying changes, monitoring errors, scheduling downtime, etc... should incur 
no more overhead with the new deployments to these TVs.

This article details some of the challenges we faced and approaches we adopted.


## Managing Environment (TV vs. Desktop)
It was necessary for our frontend to know whether it was being rendered on 
someone's desktop or if it was being displayed on one of the large TVs.

If the dashboard is being rendered for a TV we adjust behavior of the app. For
example we may change the data polling mechanism/frequency, aspects about the
app layout/styling (unfortunately CSS breakpoints could fulfill all needs here),
and in other ways detailed throughout this article.

We considered 2 options for this:

### 1. URL Query Parameter: `?isStaticDeployment=true`
We could define a url query parameter that indicates whether this app was opened
on a TV, and make sure that when we configure the TVs that this query parameter
is set.

In our React app we can use `BrowserRouter` hooks to create a custom
hook to inform our components if we're rendering on a tv or not:
```typescript
import { useLocation } from 'react-router-dom';

const IS_STATIC_DEPLOYMENT_QUERY_PARAMETER = 'isStaticDeployment';

const useIsStaticDeployment = () =>{
  const {search} = useLocation();
  const qps = new URLSearchParams(search)
  const searchQp = search.get(IS_STATIC_DEPLOYMENT_QUERY_PARAMETER)
  return searchQp === null ? false : searchQp.toLowerCase() === 'true'
}

const MyComponent = () =>{
  const isStaticDeployment = useIsStaticDeployment()
  // ...
}
```

### 2. A Separate Build for TVs
For infrastructure reasons beyond the scope of this article, we needed to have a
separate build and deployment for TV deployments. `isStaticDeployment` can be
specificed as a build argument via an environment variable, for example using 
`env-cmd`:

`package.json`
```json
{
  "scripts": {
    "build": "react-scripts build",
    "buildTv": "env-cmd -e production_tv yarn build",
    "build": "env-cmd -e production yarn build"
  }
}
```
`.env-cmdrc.json`
```json
{
  "production":{
    "REACT_APP_IS_STATIC_DEPLOYMENT": "false"
  },
  "production_tv":{
    "REACT_APP_IS_STATIC_DEPLOYMENT": "true"
  },
}
```
Then in your react app - `buildEnv.ts`
```typescript
const IS_STATIC_DEPLOYMENT = (
  process.env.REACT_APP_IS_STATIC_DEPLOYMENT?.toLowerCase() === "true" ?? false
)
```
We can then use `IS_STATIC_DEPLOYMENT` as a plain boolean `const`, or could 
create a dummy hook returning this which would make it easier for us to switch
to a more dynamic way of specifying `IS_STATIC_DEPLOYMENT` in the future (e.g
if we ever wanted to switch this to the URL param method in the future).

Option 2 is not ideal as we incur additional overhead for the new deployment of
this app and new build config.

## Managing Deployments
When a new code update is pushed for this application, the deployments on TVs
need a mechanism to be informed that there are new changes. Otherwise, these
displays may only update once every time they were manually refreshed (which 
could be days/weeks). 
When a new change is detected the TV should refresh itself, fetching the latest
bundle.

<div style="background-color:light-grey;padding: 10px;">
<small>
(Note this is a slightly different topic than cache invalidation - we aren't 
dealing with browser caching here - more just how to inform a display that never
gets reloaded manually to reload itself)
</small>
</div>
<br>

To achieve this we did the following:
### 1. Version-stamp each React deployment bundle
A simple way to do this is to write a git hash to 2 places during our build 
process:
1. A json file served alongside our app bundle
2. A `REACT_APP_` env var that we read in as a variable in react (used later).

We can write the current `HEAD`'s commit ref to a json file with:
```bash
echo '{\"build\":\"'`git rev-parse HEAD`'\"}' > build/meta.json
```

And then included it in our `package.json` build command:
`package.json`
```json
{
  "scripts": {
    "writeBuildHash": "mkdir -p build && echo '{\"build\":\"'`git rev-parse HEAD`'\"}' > build/meta.json",
    "build": "yarn writeBuildHash && export REACT_APP_BUILD_VERSION=`git rev-parse HEAD` && react-scripts build",
  }
}
```

After running `yarn build` there should be a file `build/meta.json` that can be
served alongside your app bundle that contains the following json:
```json
{"build": "<git ref hash of HEAD>"}
```
We can also access the build hash the current running deployment is using:
```typescript
const BUILD_VERSION = process.env.REACT_APP_BUILD_VERSION;
```

With these both in place, we can compare our `BUILD_VERSION` `const` with the 
most recent hash which we can get by fetching `meta.json`.

### 2. Configure the app to check for new deployments
We can do this by creating a custom React hook. It should poll for the meta.json
file and compare it to the build version baked into the current bundle (ie. 
`BUILD_VERSION`).

Tangibly:
```typescript
const BUILD_VERSION = process.env.REACT_APP_BUILD_VERSION;


```

## downtime banners
## Polling for Data
## URL state
- encode everything stateful that may vary between units in the URL
## styling (fonts and colors)
- don't use vw/vh (unless you really have to)
- use zoom instead
- theming goes a long way: light/dark mode is important.
## auth
- IP whitelisting
- credential refresh
## privacy
## auto-scroll
## polling
- web sockets
- server-sent events
- polling
- differs between your desktop environment and tv:
  - tv probably always polling
  - desktop maybe either a timeout or 'not in view' detection
## fibonnaci downtime spiral
## auto-login/launch
## launching in split-screen: a better way in future
