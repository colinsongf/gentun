# gentun: genetic algorithm for hyperparameter tuning

The purpose of this project is to provide a simple framework for hyperparameter tuning of machine learning models such
as Neural Networks and Gradient Boosted Trees using a genetic algorithm. Measuring the fitness of an individual of a
given population implies training the machine learning model using a particular set of parameters which define the
individual's genes. This is a time consuming process, therefore, a master-workers approach is used to allow several
clients (workers) perform the model fitting and cross-validation of individuals passed by a server (master). Offspring
generation by reproduction and mutation is handled by the server.

*"Parameter tuning is a dark art in machine learning, the optimal parameters of a model can depend on many scenarios."*
~ XGBoost's Notes on Parameter Tuning

# Supported models (work in progress)

- [x] XGBoost regressor
- [x] XGBoost classifier
- [ ] Scikit-learn Multilayer Perceptron Regressor
- [ ] Scikit-learn Multilayer Perceptron Classifier
- [ ] Keras

# Sample usage

## Single machine

The genetic algorithm can be run on a single box, as shown in the following example:

```python
import pandas as pd
from gentun import GeneticAlgorithm, Population, XgboostIndividual
```

```python
# Load features and response variable from train set
data = pd.read_csv('../tests/wine-quality/winequality-white.csv', delimiter=';')
y_train = data['quality']
x_train = data.drop(['quality'], axis=1)
```

```python
# Generate a random population
pop = Population(XgboostIndividual, x_train, y_train, size=100, additional_parameters={'nfold': 3})
# Run the algorithm for ten generations
ga = GeneticAlgorithm(pop)
ga.run(10)
```

You can also add custom individuals to the population before running the genetic algorithm if you already have an
intuition of which hyperparameters work well with your model. Moreover, a whole set of individuals taken from a grid
search approach could be used as the initial population. An example of how to add a customized individual is the the
following one:

```python
# Best known parameters so far
custom_genes = {
    'eta': 0.1, 'min_child_weight': 1, 'max_depth': 9,
    'gamma': 0.0, 'max_delta_step': 0, 'subsample': 1.0,
    'colsample_bytree': 0.9, 'colsample_bylevel': 1.0,
    'lambda': 1.0, 'alpha': 0.0, 'scale_pos_weight': 1.0
}
# Generate a random population and add a custom individual
pop = Population(XgboostIndividual, x_train, y_train, size=99, additional_parameters={'nfold': 3})
pop.add_individual(XgboostIndividual(x_train, y_train, genes=custom_genes, nfold=3))
```

## Multiple boxes

You can speed up the algorithm by using several machines. One of them will act as a *master*, generating a population
and running the genetic algorithm. Each time the *master* needs to evaluate an individual, it will send a request to a
pool of *workers*, which receive the model's hyperparameters from the individual and perform model fitting using n-fold
cross-validation. The more *workers* you use, the faster the algorithm will run.

First, you need to setup a [RabbitMQ](https://www.rabbitmq.com/download.html) message broker server. It will handle
communications between the *master* and all the *workers* via a queueing system.

```bash
$ sudo apt-get install rabbitmq-server
```

Start the message server and add a user with privileges to communicate the master and worker nodes. The default guest
user can only be used to access RabbitMQ locally, so the first time you start it, you should add a new user and set its
privileges as shown below:

```bash
$ sudo service rabbitmq-server start
$ sudo rabbitmqctl add_user <username> <password>
$ sudo rabbitmqctl set_user_tags <username> administrator
$ sudo rabbitmqctl set_permissions -p / <username> ".*" ".*" ".*"
```

Next, start the worker nodes. Each node has to have access to the train data. You can use as many nodes as desired as
long as they have network access to the message broker server.

```python
from gentun import GentunWorker, XgboostModel
import pandas as pd

data = pd.read_csv('../tests/wine-quality/winequality-white.csv', delimiter=';')
y = data['quality']
x = data.drop(['quality'], axis=1)

gw = GentunWorker(
    XgboostModel, x, y, host='<rabbitmq_server_ip>',
    user='<username>', password='<password>'
)
gw.work()
```

Finally, run the genetic algorithm but this time with a *DistributedPopulation* which acts as the *master* node sending
job requests to the *workers* each time an individual needs to be evaluated.

```python
from gentun import GeneticAlgorithm, DistributedPopulation, XgboostIndividual

population = DistributedPopulation(
    XgboostIndividual, size=100, additional_parameters={'nfold': 3},
    host='<rabbitmq_server_ip>', user='<username>', password='<password>'
)
# Run the algorithm for ten generations using worker nodes to evaluate individuals
ga = GeneticAlgorithm(population)
ga.run(10)
```

# References

## Genetic algorithms

* Artificial Intelligence: A Modern Approach. 3rd edition. Section 4.1.4
* https://github.com/DEAP/deap
* http://www.theprojectspot.com/tutorial-post/creating-a-genetic-algorithm-for-beginners/3

## XGBoost parameter tuning

* http://xgboost.readthedocs.io/en/latest/parameter.html
* http://xgboost.readthedocs.io/en/latest/how_to/param_tuning.html
* https://www.analyticsvidhya.com/blog/2016/03/complete-guide-parameter-tuning-xgboost-with-codes-python/

## Master-Workers model and RabbitMQ

* https://www.rabbitmq.com/tutorials/tutorial-six-python.html
