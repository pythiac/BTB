<p align="left">
<img width=15% src="https://dai.lids.mit.edu/wp-content/uploads/2018/06/Logo_DAI_highres.png" alt=“BTB” />
<i>An open source project from Data to AI Lab at MIT.</i>
</p>

![](https://raw.githubusercontent.com/HDI-Project/BTB/master/docs/_static/BTB-Icon-small.png)

A simple, extensible backend for developing auto-tuning systems.

[![PyPi](https://img.shields.io/pypi/v/baytune.svg)](https://pypi.python.org/pypi/baytune)
[![Travis](https://travis-ci.org/HDI-Project/BTB.svg?branch=master)](https://travis-ci.org/HDI-Project/BTB)
[![CodeCov](https://codecov.io/gh/HDI-Project/BTB/branch/master/graph/badge.svg)](https://codecov.io/gh/HDI-Project/BTB)

# Overview

Bayesian Tuning and Bandits is a simple, extensible backend for developing auto-tuning systems such as AutoML systems. It is currently being used in [ATM](https://github.com/HDI-Project/ATM) (an AutoML system that allows tuning of classifiers) and MIT's system for the DARPA [Data driven discovery of models program](https://www.darpa.mil/program/data-driven-discovery-of-models).

- Free software: MIT license
- Documentation: https://hdi-project.github.io/BTB
- Homepage: https://github.com/hdi-project/BTB

# Installation

## Requirements

**BTB** has been developed and tested on [Python 3.5, 3.6 and 3.7](https://www.python.org/downloads/)

Also, although it is not strictly required, the usage of a
[virtualenv](https://virtualenv.pypa.io/en/latest/) is highly recommended in order to avoid
interfering with other software installed in the system where **BTB** is run.

These are the minimum commands needed to create a virtualenv using python3.6 for **BTB**:

```bash
pip install virtualenv
virtualenv -p $(which python3.6) btb-venv
```

Afterwards, you have to execute this command to have the virtualenv activated:

```bash
source btb-venv/bin/activate
```

Remember about executing it every time you start a new console to work on **BTB**!

## Install using Pip

After creating the virtualenv and activating it, we recommend using
[pip](https://pip.pypa.io/en/stable/) in order to install **BTB**:

```bash
pip install btb
```

This will pull and install the latest stable release from [PyPi](https://pypi.org/).

## Install from Source

Alternatively, with your virtualenv activated, you can clone the repository and install it from
source by running `make install` on the `stable` branch:

```bash
git clone git@github.com:HDI-Project/BTB.git
cd BTB
git checkout stable
make install
```

## Install for Development

If you want to contribute to the project, a few more steps are required to make the project ready
for development.

First, please head to [the GitHub page of the project](https://github.com/HDI-Project/BTB)
and make a fork of the project under you own username by clicking on the **fork** button on the
upper right corner of the page.

Afterwards, clone your fork and create a branch from master with a descriptive name that includes
the number of the issue that you are going to work on:

```bash
git clone git@github.com:{your username}/BTB.git
cd BTB
git branch issue-xx-cool-new-feature master
git checkout issue-xx-cool-new-feature
```

Finally, install the project with the following command, which will install some additional
dependencies for code linting and testing.

```bash
make install-develop
```

Make sure to use them regularly while developing by running the commands `make lint` and `make test`.

# Quickstart

This section is a short series of tutorials to help you getting started with BTB.

## Tuners

Tuners are specifically designed to speed up the process of selecting the
optimal hyper parameter values for a specific machine learning algorithm.

`btb.tuning` defines Tuners: classes with a fit/predict/propose interface for suggesting sets of hyperparameters.

This is done by following a Bayesian Optimization approach and iteratively:

* letting the tuner propose new sets of hyper parameter
* fitting and scoring the model with the proposed hyper parameters
* passing the score obtained back to the tuner

At each iteration the tuner will use the information already obtained to propose
the set of hyper parameters that it considers that have the highest probability
to obtain the best results.

In order to use a tuner we will create a ``Tuner`` instance indicating which parameters
we want to tune, their types and the range of values that we want to try.

``` python
>>> from btb.tuning import GP
>>> from btb import HyperParameter, ParamTypes
>>> tunables = [
... ('n_estimators', HyperParameter(ParamTypes.INT, [10, 500])),
... ('max_depth', HyperParameter(ParamTypes.INT, [3, 20]))
... ]
>>> tuner = GP(tunables)
```

Then we perform the following three steps in a loop.

1. Let the Tuner propose a new set of parameters

    ``` python
    >>> parameters = tuner.propose()
    >>> parameters
    {'n_estimators': 297, 'max_depth': 3}
    ```

2. Fit and score a new model using these parameters

    ``` python
    >>> model = RandomForestClassifier(**parameters)
    >>> model.fit(X_train, y_train)
    RandomForestClassifier(bootstrap=True, class_weight=None, criterion='gini',
                max_depth=3, max_features='auto', max_leaf_nodes=None,
                min_impurity_decrease=0.0, min_impurity_split=None,
                min_samples_leaf=1, min_samples_split=2,
                min_weight_fraction_leaf=0.0, n_estimators=297, n_jobs=1,
                oob_score=False, random_state=None, verbose=0,
                warm_start=False)
    >>> score = model.score(X_test, y_test)
    >>> score
    0.77
    ```

3. Pass the used parameters and the score obtained back to the tuner

    ``` python
    tuner.add(parameters, score)
    ```

At each iteration, the `Tuner` will use the information about the previous tests
to evaluate and propose the set of parameter values that have the highest probability
of obtaining the highest score.

For more detailed examples, check scripts from the `examples` folder.



## Selectors

Selectors apply multiple strategies to decide which models or families of models to
train and test next based on how well thay have been performing in the previous test runs.
This is an application of what is called the Multi-armed Bandit Problem.

`btb.selection` defines Selectors: classes for choosing from a set of discrete options with multi-armed bandits.

The process works by letting know the selector which models have been already tested
and which scores they have obtained, and letting it decide which model to test next.

The selectors are intended to be used in combination with tuners in order to find
out and decide which model seems to get the best results once it is properly fine tuned.

In order to use the selector we will create a ``Tuner`` instance for each model that
we want to try out, as well as the ``Selector`` instance.

```
>>> from sklearn.ensemble import RandomForestClassifier
>>> from sklearn.svm import SVC
>>> models = {
...     'RF': RandomForestClassifier,
...     'SVC': SVC
... }
>>> from btb.selection import UCB1
>>> selector = UCB1(['RF', 'SVC'])
>>> tuners = {
...     'RF': GP([
...         ('n_estimators', HyperParameter(ParamTypes.INT, [10, 500])),
...         ('max_depth', HyperParameter(ParamTypes.INT, [3, 20]))
...     ]),
...     'SVC': GP([
...         ('c', HyperParameter(ParamTypes.FLOAT_EXP, [0.01, 10.0])),
...         ('gamma', HyperParameter(ParamTypes.FLOAT, [0.000000001, 0.0000001]))
...     ])
... }
```

Then we perform the following steps in a loop.

1. Pass all the obtained scores to the selector and let it decide which model to test.

    ``` python
    >>> next_choice = selector.select({'RF': tuners['RF'].y, 'SVC': tuners['SVC'].y})
    >>> next_choice
    'RF'
    ```

2. Obtain a new set of parameters from the indicated tuner and create a model instance.

    ``` python
    >>> parameters = tuners[next_choice].propose()
    >>> parameters
    {'n_estimators': 289, 'max_depth': 18}
    >>> model = models[next_choice](**parameters)
    ```

3. Evaluate the score of the new model instance and pass it back to the tuner

    ``` python
    >>> model.fit(X_train, y_train)
    RandomForestClassifier(bootstrap=True, class_weight=None, criterion='gini',
                max_depth=18, max_features='auto', max_leaf_nodes=None,
                min_impurity_decrease=0.0, min_impurity_split=None,
                min_samples_leaf=1, min_samples_split=2,
                min_weight_fraction_leaf=0.0, n_estimators=289, n_jobs=1,
                oob_score=False, random_state=None, verbose=0,
                warm_start=False)
    >>> score = model.score(X_test, y_test)
    >>> score
    0.89
    >>> tuners[next_choice].add(parameters, score)
    ```

## What's next?

For more details about **BTB** and all its features, please check the
[documentation site](https://HDI-Project.github.io/BTB/).


# Credits

BTB is an open source project from the Data to AI Lab at MIT which has been built and maintained
over the years by the following team:

* Carles Sala <csala@csail.mit.edu>
* Micah Smith <micahs@mit.edu>
* Laura Gustafson <lgustaf@mit.edu>
* Bennett Cyphers <bennettcyphers@gmail.com>
* Kalyan Veeramachaneni <kalyan@csail.mit.edu>
* Alfredo Cuesta-Infante <alfredo.cuesta@urjc.es>
* Plamen Valentinov <plamen@pythiac.com>
* Sze Nga Wong <wsnalice@mit.edu>


## Citing ATM

If you use BTB, please consider citing the following work:

- Laura Gustafson. Bayesian Tuning and Bandits: An Extensible, Open Source Library for AutoML. Masters thesis, MIT EECS, June 2018. [(pdf)](https://dai.lids.mit.edu/wp-content/uploads/2018/05/Laura_MEng_Final.pdf)

BibTeX entry:

```bibtex
  @MastersThesis{Laura:2018,
    title = "Bayesian Tuning and Bandits: An Extensible, Open Source Library for AutoML",
    author = "Laura Gustafson",
    month = "May",
    year = "2018",
    url = "https://dai.lids.mit.edu/wp-content/uploads/2018/05/Laura_MEng_Final.pdf",
    type = "M. Eng Thesis",
    address = "Cambridge, MA",
    school = "Massachusetts Institute of Technology",
  }
```

# Related Projects

* **ATM**: https://github.com/HDI-Project/ATM
* **AutoBazaar**: https://github.com/HDI-Project/AutoBazaar
* **mit-d3m-ta2**: https://github.com/HDI-Project/mit-d3m-ta2
