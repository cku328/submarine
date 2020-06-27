<!---
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->



![color_logo_with_text](docs/assets/color_logo_with_text.png)

[![Build Status](https://travis-ci.org/apache/submarine.svg?branch=master)](https://travis-ci.org/apache/submarine) [![License](https://img.shields.io/badge/license-Apache%202-4EB1BA.svg)](https://www.apache.org/licenses/LICENSE-2.0.html)  [![HitCount](http://hits.dwyl.io/apache/submarine.svg)](http://hits.dwyl.io/apache/submarine)

# What is Apache Submarine?

Apache Submarine (Submarine for short) is the `ONE PLATFORM` to allow Data Scientists to create end-to-end machine learning workflow. `ONE PLATFORM` means it supports Data Scientists to finish their jobs on the same platform without frequently switching their toolsets. From dataset exploring data pipeline creation, model training (experiments), and push model to production (model serving and monitoring). All these steps can be completed within the `ONE PLATFORM`.

## Why Submarine?

There're already a lot of open-source and commercial projects are trying to create a end-to-end machine-learning/deep-learning platform, what's the vision of Submarine?

### Problems

1) Existing products lack of good User-Interface (API, SDK, etc) to run training workload at scale, repeatable and easy for data scientist to understand on cloud/premise.
2) Data-Scientist want to focus on domain-specific target (e.g. improve Click-Through-Rate), however available products always give user a platform (a SDK to run distributed Pytorch script).
3) Many products provided functionalities to do data exploring, model training, and serving/monitoring. However these functionalities are largely disconnected with each other. And cannot organically work with each other.

_Theodore Levitt_ once said:

```
“People don’t want to buy a quarter-inch drill. They want a quarter-inch hole.”
```

### Goals of Submarine

#### Model Training (Experiment)

- Can run experiment (training jobs) on prem, on cloud. Via easy-to-use User-Interfaces
- Easy for Data-Scientist (DS) to manage training code and dependencies (Docker, Python Dependencies, etc.) .
- ML-focused APIs to run/track experiment from Python SDK (notebook), REST API, and CLI.
- Provide APIs to run training jobs by using popular frameworks (Standalone/Distributed TensorFlow/PyTorch/Hovorod).
- Pre-packaged Training Template for Data-Scientists to focus on domain-specific tasks (like using DeepFM to build a CTR prediction model).
- Support GPU and other compute speed-up devices.
- Support running on K8s/YARN or other resource management system.
- Pipeline is also on the backlog, we will look into pipeline for training in the future.

#### Notebook Service

- Submarine is target to provide notebook service, which allows users to create/edit/delete a notebook instance (such as a Jupyter notebook) running on the cluster.
- Users can submit experiment, manage models using Submarine SDK.

#### Model Management (Serving/versioning/monitoring, etc.)

- Model management for model-serving/versioning/monitoring is on the roadmap.

## Easy-to-use User-Interface of Submarine

Like mentioned above, Submarine is targeted to bring Data-Scientist-friendly user-interfaces to make their life easier. Here're some examples of Submarine user-interfaces.

### Submit a distributed Tensorflow experiment via Submarine Python SDK

#### Run a Tensorflow Mnist experiment
```python

# New a submarine client of the submarine server
submarine_client = submarine.ExperimentClient(host='http://localhost:8080')

# The experiment's environment, could be Docker image or Conda environment based
environment = Environment(image='gcr.io/kubeflow-ci/tf-dist-mnist-test:1.0')

# Specify the experiment's name, framework it's using, namespace it will run in,
# the entry point. It can also accept environment variables. etc.
# For PyTorch job, the framework should be 'Pytorch'.
experiment_meta = ExperimentMeta(name='mnist-dist',
                                 namespace='default',
                                 framework='Tensorflow',
                                 cmd='python /var/tf_dist_mnist/dist_mnist.py --train_steps=100')
# 1 PS task of 2 cpu, 1GB
ps_spec = ExperimentTaskSpec(resources='cpu=2,memory=1024M',
                             replicas=1)
# 1 Worker task
worker_spec = ExperimentTaskSpec(resources='cpu=2,memory=1024M',
                                 replicas=1)

# Wrap up the meta, environment and task specs into an experiment.
# For PyTorch job, the specs would be "Master" and "Worker".
experiment_spec = ExperimentSpec(meta=experiment_meta,
                                 environment=environment,
                                 spec={'Ps':ps_spec, 'Worker': worker_spec})

# Submit the experiment to submarine server
experiment = submarine_client.create_experiment(experiment_spec=experiment_spec)

# Get the experiment ID
id = experiment['experimentId']

```

#### Query a specific experiment
```python
submarine_client.get_experiment(id)
```

#### Wait for finish

```python
submarine_client.wait_for_finish(id)
```

#### Get the experiment's log
```python
submarine_client.get_log(id)
```

#### Get all running experiment
```python
submarine_client.list_experiments(status='running')
```

For a quick-start, see [Submarine On K8s](docs/userdocs/k8s/README.md)


### Submit a pre-defined experiment template job

### Submit an experiment via Submarine UI

(Available on 0.6.0, see Roadmap)

## Architecture, Design and requirements

If you want to know more about Submarine's architecture, components, requirements and design doc, they can be found on [Architecture-and-requirement](docs/design/architecture-and-requirements.md)

Detailed design documentation, implementation notes can be found at: [Implementation notes](docs/design/implementation-notes.md)

## Apache Submarine Community

Read the [Apache Submarine Community Guide](./docs/community/README.md)

How to contribute [Contributing Guide](./docs/community/contributing.md)

Issue Tracking: https://issues.apache.org/jira/projects/SUBMARINE

## User Document

See [User Guide Home Page](docs/user-guide-home.md)

## Developer Document

See [Developer Guide Home Page](docs/development-guide-home.md)

## Roadmap

What to know more about what's coming for Submarine? Please check the roadmap out: https://cwiki.apache.org/confluence/display/SUBMARINE/Roadmap

## License

The Apache Submarine project is licensed under the Apache 2.0 License. See the [LICENSE](./LICENSE) file for details.
