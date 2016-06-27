[![Build Status](https://jenkins.bit.wooga.com/buildStatus/icon?job=bit.airconditioner/test)](https://jenkins.bit.wooga.com/job/bit.airconditioner/test)

# bit.airconditioner

Airconditioner is an extension library to Airbnb/Apache's Airflow tool. It allows to construct the directed acyclic graphs
(DAGs) of tasks through YAML description files. This provides a more user-friendly and less repetitive interface for creating new DAGs.
Currently designed and tested for Wooga's Airflow (custom version of Airflow).

## Setup

    pip install -r requirements.txt

To import Airconditioner from a different project, add this to its `requirements.txt`:

    -e git+https://github.com/wooga/bit.airconditioner.git@master#egg=airconditioner

## Testing
We use [Behave](http://pythonhosted.org/behave/) for BDD (Behavior Driven Development).
The tests are very close to natural English defined in `.feature` files located under the `/features` directory.
To run the tests run:

    behave

## Defining DAGs through YAMLs

### Basic Usage

From a DAG definition `py` file:

```python
# The strings "DAG" and "airflow" need to be present here, even under a comment, because
# this is how airflow identifies that a python file is a DAG definition file at the moment.
from airconditioner import DAGBuilder
DAGBuilder(yaml_path=<yaml_path>).build(target=globals())
```

### YAML path & files

Airconditioner's DAG builder takes a `yaml_path` argument, which is the location of a directory containing 4 YAML files
necessary for building the tasks DAGs:

* `games.yaml`: Where the game DAGs are defined.
* `tasks.yaml`: Where the tasks are defined.
* `clusters.yaml`: Categorizes tasks in logical groups to be referenced in `games.yaml`.
* `dependencies.yaml`: Specifies the dependency chaining of tasks.

#### Games

A minimum game configuration requires an identification, a `platform` and the default arguments `start_date` and `owner`.
Example:

```yaml
my_game_name:
  default_args:
    start_date: 2016-03-20
    owner: deploy
  platform: android
```

Optional attributes:

* default_args:
     * end_date: Date in format yyyy-mm-dd`
     * depends_on_past: `True` or `False`
     * email: List of warning emails
     * email_on_failure: `True` or `False`
     * email_on_retry: `True` or `False`
* clusters:
     * list of task clusters. Important: they need to be present in `clusters.yaml`
          * the same attributes in `default_args` can be repeated inside a cluster in case a specific cluster has different settings.
* params
     * custom parameters that will be passed ahead to tasks

Example:
```yaml
my_game_name:
  default_args:
    start_date: 2016-03-20
    end_date: 2016-04-01
    owner: deploy
    depends_on_past: True
    email:
      - bit-admin@wooga.net
    email_on_failure: True
    email_on_retry: False
    queue: consumer-jail-04
  clusters:
    bookings:
      start_date: 2016-04-25
    another_cluster:
  platform: android
  params:
    table_name: calc_something
```

It is possible to set a default DAG with default arguments and parameters there are common to all DAGs,
so they don't have to be repeated. Example:
```yaml
default:
  default_args:
    owner: deploy
```

#### Tasks
A task in `tasks.yaml` has an identification and can be defined for multiple profiles and platforms with the following structure:
```yaml
<identification>:
    <profiles>:
        <platforms>:
            <task settings>
```

It is possible to have a `default` profile and a `default` platform if the task settings are the same for multiple profiles or platforms.


The tasks settings can vary according to its type. By default, the `type` attribute is a required attributed in the task settings.
There are multiple possible types of tasks:

<!--TODO: include types descriptions, or link to airflow docs, or find another solutiongit-->

* time_sensor
* time_delta
* exasol
* sleep
* sql_sensor
* task
* subschedule
* bash
* dummy
* none

Example:
```yaml
a_task_to_calc_something:
  default:
    ios:
      type: sql_sensor
      conn_id: exasol
      sql: SELECT * from table;
  low_traffic:
    default:
      type: time_delta
      delta: !timedelta 2h
```

#### Clusters

Because some tasks are often present together in some DAGs, we have clusters. Clusters organize tasks in logical groups,
so they can be referenced in `games.yaml` easier, instead of having to add tasks one by one. A cluster requires an identification
to be referenced and the list of tasks present in the cluster. These tasks need to be previously defined in `tasks.yaml`.
It is possible to add some configurations to each task.


Important: A task can be present in more than one cluster

Example:
```yaml
my_cluster_name:
 - a_task_to_calc_something
 - another_task_to_calc_something_else:
     start_date: 2016-06-21
```

#### Dependencies

In Airflow, tasks are organized in directed acyclic graphs, which means that some tasks are chained after the other
without creating loops. In `dependencies.yaml`, you can define which tasks are required for a task to run with the following
 structure:

```yaml
<task>:
    - <dependency>
    - <dependency>
    - <dependency>
    - (<optional-dependency>)
    - (<optional-dependency>)
```
These tasks need to be previously defined in `tasks.yaml`. Optional dependencies for task, which might only be added to
the DAG in certain cases can be defined in parenthesis (i.e. `(task_id)` instead of `task_id`) and will not throw an 
exception in case they're missing.
