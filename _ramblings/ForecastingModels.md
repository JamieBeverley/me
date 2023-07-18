---
layout: post
title: 'An Abstraction for Models that Forecast Wait Times'
toc: true
order: 4
---

# Context
With Emergency Department (ED) wait times at historic highs in Ontario due to byproducts of the pandemic, our team developed a product that displays estimated wait times for patients on large TVs in the ED.

We knew early in this project that we would have multiple competing models with tradeoffs to coverage (% of patients who are seen within the wait time we predict) and accuracy (e.g. mean error of predicted wait times to actual wait times).

In developing the application integration for this product we anticipated the following requirements:
- We want to run multiple models (types, versions) in parallel
- We want to easily be able to switch between models if a challenger model is performing better than what we have in production
- We want models and their predictions to be version-controlled so we can always tell which model produced which prediction/forecast
- Updating a model version or adding a new model should require minimal engineering, focused solely on modeling (and not, for example, reading/writing to DBs, scheduling, or how predictions make it to the frontend).

# Implementation
We started by encoding the requirements of a model in a python interface/abstract base classe (ABC).
A model in this conception:
- must have a name and version associated to it (together, uniquely identify a model)
- must define a function for productin a prediction, given some input data type `DataT`

```python
import pydantic, Extra
DataT = TypeVar('DataT')

# For now, a 'wait prediction' is some number of seconds we expect someone to 
# wait. This could grow to be a much more rich data type though.
class Prediction:
  
  class Config:
    extra = Extra.forbid
  
  wait_seconds: float


class Model(ABC):
  
  # All models should specify a model name  
  @property
  @abstractmethod
  def name(self) -> str:
    raise NotImplementedError

  # All models should have a version associated  
  @property
  @abstractmethod
  def version(self) -> str:
    raise NotImplementedError

  @classmethod
  @abstractmethod
  def predict(cls, data: DataT) -> Prediction:
    raise NotImplementedError
```
Lets assume that our `DataT` is a `pandas.DataFrame` object containing wait times for the last 2 weeks.
A concrete class implementing this interface for a very basic moving average model would look something like this:

```python
class MovingAverageModel(Model):
  name = "moving_average"
  version = "0.0.1"

  @classmethod
  def predict(cls, data: pd.DataFrame) -> Prediction:
    avg = data["wait_time_seconds"].mean()
    return Prediction(wait_seconds=avg)
```

We probably want to specify a bounds for the moving average though (E.g. only average wait times from the last 2 hours). This behavior can be encapsulated in another abstract class so child classes can specify their own time-window:
```python
class MovingAverageModelABC(ABC, Model):
  version = "0.0.1"
  
  @property
  @abstractmethod
  def lookback_seconds(self) -> float:
    raise NotImplementedError

  @classmethod
  def predict(cls, data: pd.DataFrame) -> Prediction:
    since = datetime.utcnow() - timedelta(seconds=cls.lookback_seconds)
    avg = data[data.disposition_ts > since]["wait_seconds"].mean()
    return Prediction(wait_seconds=avg)
```
Then we can specify different variants of this model easily in a child class:
```python
class MovingAverageModel120(MovingAverageModel):
  name = "moving_average_120"
  lookback_seconds = 60*60*2

class MovingAverageModel60(MovingAverageModel):
  name = "moving_average_60"
  lookback_seconds = 60*60

# etc...
```
We extended this class hierarchy to a few other types of models in python (ETS, ARIMA) so we can create new models and parameterize models differently:
```python
class ETSModelX(ETSModel):
  seasonal = 'add'
  error = 'add'

class ARIMAModelX(ARIMAModel):
  lag = 5
  q = 0.9  
```

# Models in Other Languages (via HTTP API)
So far, all models here are implemented in python, in one big package.

This is not ideal: what if different models had different requirements (e.g. required different versions of `statsmodels`), or if we wanted to deploy a model that was not python at all?

It made sense to extend the simple class structure to enable more isolated and flexible model deployments.

We chose to do this via an HTTP interface. A new type of model could be specified as one that is hosted as an HTTP API and conforming to an interface that is expected by the class structure.

```python
class HTTPAPIModel(ABC, Model):
  # maintain private internal name/version. Ultimately these will come from the
  # model API but we can cache them in memory to avoid unecessary http calls.
  # Note: should probably use an actual cache w/ an appropriate timeout in 
  #    production. If this were used in a long running process this may report 
  #    a stale name/value 
  _name:Union[str, None] = None
  _version:Union[str, None] = None

  # All models over HTTP API must define an endpoint root
  # eg. https://example.com/api/v1
  @property
  @abstractmethod
  def api_root(self) -> str:
    raise NotImplementedError

  @classmethod
  def _fetch_model_metadata(cls) -> None:
    endpoint = f"{cls.api_root}/metadata"
    response = requests.get(endpoint, timeout=10)
    response.raise_for_status()
    cls._name = response["name"]
    cls._version = response["version"]

  # It really should be up to the model code to define its name and version
  # (rather than trying to do so here). Model metadata should live alongside
  # model code so maintainers of models can update these alongside updates to
  # the model. Therefore, we pull these via a /metadata endpoint via a private
  # method self._fetch_model_metadata
  @property
  def name(self):
    if self._name is None:
      self._fetch_model_metadata()    
    return self._name
  
  @property
  def version(self):
    if self._version is None
      self._fetch_model_metadata()
    return self._version
  
  @classmethod
  def predict(cls, data: pd.DataFrame) -> Prediction:
    endpoint = f"{cls.api_root}/predict"
    response = requests.post(
      endpoint,
      json=data.to_json(orient="records")
      timeout=30
    )
    response.raise_for_status() 
    return Prediction(
      wait_time_seconds=response.json['wait_time_seconds']
    )
```

# Integration
At this point we have some number of models conforming to an interface 
specification.

We could integrate this into a frontend by including source code directly or by
packaging this as a python package and importing it in frontend applications
code but this is not ideal: we would be baking a lot of model-specific 
dependencies into a frontend repository. Unless we're somewhat careful managing
processes/threads we could accidentally tie up processes with running
inference/prediction instead of serving workloads required for the UI.

Instead, we choose to run models via a scheduler every `X` minutes:
- model execution job runs every 15 mins, executing all models
- predictions are written to a database
- frontend application can pull the most recent predictions from this database
- an internal monitoring tool can pull recent and historical predictions to
monitor performance and drift.

## Batch Model Runs
We just need some utilities to do the following at a regular interval:
- pull our input data from a database
- iterate through our model classes and run inference
- save model predictions to a database
- alert us of any errors if a model fails

A dumb implementation in pseudo-code:

```python
def get_input_df(con:Connection) -> pd.DataFrame:
  # select some input data from a database source
  raise NotImplementedError

def save_prediction(con: Connection, model:Model, prediction:Prediction) -> None:
  # save a prediction w/ the model name and model version
  raise NotImplementedError

def alert_errors(errors: List[Exception]) -> None:
  # log the error, eg. via Sentry or anywhere else you will be alerted
  raise NotImplementedError


def main():
  con = engine.create("...")
  input_df = get_input_df(con)
  errors = []
  
  # Iterate through each model, generate a prediction, save it to database.
  # Naive implementation with a for-loop. Could be parallelized via threads or
  # processes if needed.
  for model in models:
    try:
      prediction = model.predict(input_df)
      save_prediction(con, model, prediction)
    except Exception as exc:
      erros.append(exc)
  if errors:
    try:
      alert_errors(errors)
    finally:
      raise Execption(f"{len(errors)} models failed to run") from errors[0]
```
It might be tedious to have to export models everywhere and import them into 
this main script. We could add a simple `staticmethod` helper to recurse through
the class hierarchy and provide a list of 'concrete' model classes:
```python
class Model(ABC):
  # ...
  
  @staticmethod
  def get_concrete_models(cls=None) -> List[Model]:
    if cls is None:
      cls = Model
    classes = [c for c in cls.__subclasses__()]
    for c in classes:
      classes.extend(Model.get_concrete_models(c))
    return  [c for c in classes if ABC not in c.__bases__]

# ...

def main():
  con = engine.create("...")
  input_df = get_input_df(con)
  errors = []
  
  # Now we only need to import the Model class here...
  for model in Model.get_concrete_models():
     try:
      prediction = model.predict(input_df)
      save_prediction(con, model, prediction)
    except Exception as exc:
      erros.append(exc)
  if errors:
    try:
      alert_errors(errors)
    finally:
      raise Execption(f"{len(errors)} models failed to run") from errors[0]
```

We run something like this every 15 mins or so, scheduled as a juypter notebook
(but could be implemented via another scheduler; crontab, Airflow, etc...).

## Frontend Integration
Our frontend for this tool is a Flask backend API and a React SPA.

The Flask backend defines a simple `GET` endpoint for retrieving predictions
from the predictions DB. We use a config within Flask to define the default
model we serve predictions for, and allow the client to override these by 
specifying a model name/version through query parameters:
```python 
from flask import request, current_app, Response

# ...

@app.route("/wait_time", methods=["GET"])
def wait_time():
  config = current_app.config

  # Parse query params
  model_name = request.args.get("model_name", config["WAIT_TIMES_MODEL_NAME"])
  model_version = request.args.get(
    "model_version",
    config["WAIT_TIMES_MODEL_VERSION"]
  )

  to_ts = datetime_from_epoch(to_ts) if 'to_ts' in request.args else datetime.utcnow()
  from_ts = datetime_from_epoch(from_ts) if 'from_ts' in request.args else to_ts - timedelta(hours=6)
  
  # Fetch Prediction
  predictions = get_most_recent_predictions(
    con=engine.connect(),
    model_name=model_name,
    model_version=model_version,
    from_ts=from_ts,
    to_ts=to_ts,
  )
  
  return Response(predictions.json, status=200)
```

- By setting a default model via Flask config, we can manage which model results
are shown to end-users without making code changes (just changes to config).
This makes it a bit easier to switch to a different model if we publish a new
model and its performing better than the current default. In our practice,
this config ultimately comes from 2 environment variables which can be adjusted
on a deployment without deploying code changes.
- This API can be leveraged for 2 frontends:
  - frontend for end-users can query this api with no parameters
  - we can use this API for a model-monitoring frontend (to observe things like
  model drift). For this use case we probably need to make use of the query
  parameters to override defaults (`from_ts`, `to_ts`, `model_name`,
  `model_version`)
