---
layout: post
title: 'Developing Dashboards for Passive Displays'
toc: true
modified_date: false
order: 4
---

{:toc}

## Context
Developing an application that runs on a passive display in a shared space can
have additional complexity.

We needed to deploy an existing React/Redux data dashboard on 40 TVs of
varying resolution accross our organization.
A summary of some requirements:
- Must update every minute.
- May be displayed in some public areas, in which sensitive data.
must be masked/obfuscated for privacy reasons
- Must automatically launch when the PC connected to the TV turns.
on
- We'd like to minimize additional overhead in maintaining the TVs (obviously).
Deploying changes, monitoring errors, scheduling downtime, etc... should incur 
no more overhead with the new deployments to these TVs.

This article details some of the challenges we faced and approaches we adopted.

## Managing Environment (TV vs. Desktop)
It was necessary for our frontend to know whether it was being rendered on 
someone's desktop or if it was being displayed on one of the large TVs.

If the dashboard is being rendered for a TV we adjust behavior of the app. For
example we may change the data polling mechanism/frequency, aspects about the
app layout/styling (unfortunately CSS breakpoints could not fulfill all needs here),
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
    "buildTv": "env-cmd -e tv yarn build",
    "buildProduction": "env-cmd -e production yarn build"
  }
}
```
`.env-cmdrc.json`
```json
{
  "production":{
    "REACT_APP_IS_STATIC_DEPLOYMENT": "false"
  },
  "tv":{
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

const useNewDeploymentAvailable = (intervalSeconds: number):boolean => {
  const [newAvailable, setNewAvailable] = useState(false)
  
  useEffect(() => {
    // Abort controller for aborting api request (cleanup)
    const abortController = new AbortController();
    
    // Interval for polling for changes to meta.json
    const interval = setInterval(async () => {
      const meta = await fetch(
        "/meta.json",
        {
          // Make sure we're not just getting a cached version of meta.json
          headers:{"cache-control": "no-cache"},
          signal: abortController.signal
        }
      )
      const json = await meta.json()
      setNewAvailable(json.build !== BUILD_HASH)
    }, intervalSeconds*1000)
    return () => {
      clearInterval(interval)
      abortController.abort()
    }
  }, [intervalSeconds])
  
  return newAvailable
}

const useRefreshOnNewDeployment = () => {
  const newDeployment = useNewDeploymentAvailable(60);
  useEffect(() => {
    if (newDeployment) {
      window.location.reload()
    }
  }, [newDeployment])
}
```
We can use this somewhere high in our component hierarchy:
```typescript
const App = () =>{
  useRefreshOnNewDeployment();
  return (
    //...
  )
}
```

## Recovering From Catastrophic Error Boundaries
If a frontend error occurs and we hit an 'everything is broken' [error boundary](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary)
high up in our component hierarchy then we need a way to attempt to recover from
that error without requiring someone to manually refresh.

Again, folks generally aren't interacting w/ these passive displays and it would
be a nightmare for the dev team to have to remote into a bunch of PCs to refresh
following a downtime.

More thought complete, this error boundary should:
- Alert us so we're aware an error occured
- Render an appropriate error message
- Attempt to recover by reloading the page

We also don't want to reload too quickly/often in the event of an error.

A naiive implementation might just try to refresh the page immediately on every
error. But if the error is the result of some backend or infra issue it would
likley be detrimental to then flood the backend/web server that is serving our
React bundle with even more requests.

Instead we can introduce some timeout before attempting to refresh, and
increment that timeout for each successive failure/error.

So the process should be:
- Error boundary caught
- Report the error
- Render an error message
- Set a timeout to refresh the page
- Refresh the page with a url parameter indicating the next timeout duration

In code:
```typescript
type Props = {
  // Callback for reporting an error should it occur. We assume we're
  // reporting errors via some HTTP call (returning an aysnc Response)
  reportError: (error: Error) => Promise<Response>;
  children?: React.ReactNode;
}

type State = {
  // Have we caught an error?
  hasError: boolean;
  // Has the error been successfully reported?
  errorReported: boolean;
}

export class ErrorBoundary extends Component<Props, State> {
  // The name of the URL parameter we'll use to determine how long to wait
  // before refreshing the page when an error occurs.
  private readonly urlParameter = 'refreshDurationSeconds'
  // Default amount of time to wait after first error.
  private readonly defaultRefreshDurationSeconds = 5;
  // Internal reference to a timeout. We'll use this delay some number of
  // seconds before trying to refresh in the case of an error
  private refreshTimeout: null | ReturnType<typeof setTimeout> = null;
  // Timeout for reseting the URL parameter once the app has recovered
  private clearRefreshTimeout: null | ReturnType<typeof setTimeout> = null;

  state = {hasError: false, errorReported: false}

  private getRefreshDurationSeconds(): number {
    // Parse the URL parameter containing how long to wait before refreshing
    // after an error - default to some number of seconds.
    const urlParameters = new URLSearchParams(window.location.search);
    const refreshDuration = urlParameters.get(this.urlParameter)
    if (refreshDuration) return parseInt(refreshDuration)
    return this.defaultRefreshDurationSeconds
  }

  private reload(duration: number) {
    // Reload the page, setting the URL parameter to the new duration we'll wait
    // next time, if an error occurs.
    const urlParameters = new URLSearchParams(window.location.search);
    urlParameters.set('refreshDurationSeconds', duration.toString())
    window.location.href = `${window.location.pathname}?${urlParameters}`
  }

  public static getDerivedStateFromError(): State {
    return {hasError: true, errorReported: false}
  }

  async componentDidCatch(error: Error) {
    // If we catch an error, report it and set the state so the render method
    // can inform the user that it has been reported
    const response = await this.props.reportError(error);
    this.setState({errorReported: response.ok})
  }

  componentDidUpdate(prevProps: Readonly<Props>, prevState: Readonly<State>, snapshot?: any): void {

    // If a new error occurred, wait the specified duration, and then refresh.
    if (this.state.hasError && !prevState.hasError) {
      const waitDuration = this.getRefreshDurationSeconds()
      this.refreshTimeout = setTimeout(() => {
        // We reload, setting the next reload duration to be 1.5x the current.
        // So we'll iteratively increase the wait time. We can clip this at some
        // max value (eg. 60s here).
        this.reload(Math.min(60, Math.round(waitDuration*1.5)));
      }, waitDuration * 1000)
    }
  }

  componentWillUnmount(): void {
    if (this.refreshTimeout) {
      clearTimeout(this.refreshTimeout)
    }
    if (this.clearRefreshTimeout) {
      clearTimeout(this.clearRefreshTimeout)
    }
  }

  componentDidMount(): void {
    // When this component mounts, check if the URL parameter for the error
    // refresh was set. If it is, we'll clear it some time (5s) after the
    // refresh duration.
    // That way, if we had an error on our last render and refreshing solved it,
    // then we reset the refresh duration back down to the default if an error
    // occurs later.
    const waitDuration = this.getRefreshDurationSeconds()
    this.clearRefreshTimeout = setTimeout(() => {
      const urlParameters = new URLSearchParams(window.location.search);
      if (urlParameters.has(this.urlParameter)) {
        urlParameters.delete(this.urlParameter)
        const newUrl = `${window.location.pathname}${urlParameters.size ? `?${urlParameters}` : ''}`
        window.history.replaceState(null, '', newUrl)
      }
    }, waitDuration * 1000 + 5000)
  }

  render(): ReactNode {
    if (this.state.hasError) {
      let message = "An unexpected error occurred."
      if (this.state.errorReported) {
        message = `${message} We have been notified and are working on it.`
      }
      return <div>{message}</div>
    }
    return this.props.children
  }
}
```

## Future Topics
Here are a few other topics that I'll write about eventually...

- Managing maintenance and (un)planned downtime
- Retrieving data (polling, websockets, server-sent events)
- Encoding state in URL
- Styling (fonts, colors, viewport dimensions)
- Configuring automatic authentication for machines in kiosk mode
- A privacy mode for hiding sensitive data in public areas
- Auto-scrolling tables that overflow
- Launching views in split-screen
