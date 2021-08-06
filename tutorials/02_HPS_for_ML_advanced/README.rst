.. _tutorial-02:

Hyperparameter Search for Machine Learning (Advanced)
*****************************************************

.. warning::

    Be sure to work in a virtual environment where you can easily ``pip install`` new packages. This typically entails using either Anaconda, virtualenv, or Pipenv.

In this tutorial, we will show how to treat a learning method as a hyperparameter in the hyperparameter search.
We will consider `Random Forest (RF) classifier <https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html>`_
and `Gradient Boosting (GB) classifier <https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.GradientBoostingClassifier.html>`_
methods in `scikit-learn <https://scikit-learn.org/stable/>`_ for the Airlines data set.
Each of these methods have its own set of hyperparameters and some common parameters.
We model them using `ConfigSpace  <https://automl.github.io/ConfigSpace/master/>`_
a python package to express conditional hyperparameters and more.

Let us start by creating a DeepHyper project and a problem for our application:


.. note::
    If you already have a DeepHyper project you do not need to create a new one for each problem.

.. code-block:: console
    :caption: bash

    deephyper start-project dh_project
    cd dh_project/dh_project/
    deephyper new-problem hps advanced_hpo
    cd advanced_hpo/

Create a ``mapping.py`` script where you will record the classification algorithms of interest (``touch mapping.py`` in the terminal then edit the file):

.. literalinclude:: dh_project/dh_project/advanced_hpo/mapping.py
    :language: python
    :caption: advanced_hpo/mapping.py
    :linenos:

Create a script to test the accuracy of the default configuration for both the models:

.. literalinclude:: dh_project/dh_project/advanced_hpo/default_configs.py
    :linenos:
    :caption: advanced_hpo/default_configs.py

Run the script and record the training, validation, and test accuracy as follows:

.. code-block:: console
    :caption: bash

    python default_configs.py

Running the script will give the the following outputs:

.. code-block:: python
    :caption: [Out]

    RandomForest
    Accuracy on Training: 0.879
    Accuracy on Validation: 0.621
    Accuracy on Testing: 0.620

    GradientBoosting
    Accuracy on Training: 0.649
    Accuracy on Validation: 0.648
    Accuracy on Testing: 0.649

The accuracy values show that the RandomForest classifier with default hyperparameters results in overfitting and thus poor generalization
(high accuracy on training data but not on the validation and test data).
On the contrary GradientBoosting does not show any sign of overfitting and has a better accuracy on the validation and testing set,
which shows a better generalization than RandomForest.

Next, we optimize the hyperparameters, where we seek to find the right classifier and its corresponding hyperparameters,
and improve the accuracy on the vaidation and test data.
Create ``load_data.py`` file to load and return training and validation data:

.. literalinclude:: dh_project/dh_project/advanced_hpo/load_data.py
    :linenos:
    :caption: advanced_hpo/load_data.py

.. note::
    Subsampling with ``X_train, y_train = resample(X_train, y_train, n_samples=int(1e4))`` can be useful if you want to speed-up your search. By subsampling the training time will reduce.

To test this code:

.. code-block:: console
    :caption: bash

    python load_data.py

The expected output is:

.. code-block:: python
    :caption: [Out]

    X_train shape: (10000, 7)
    y_train shape: (10000,)
    X_valid shape: (10000, 7)
    y_valid shape: (10000,)

Create ``model_run.py`` file to train and evaluate a given hyperparameter configuration.
This function has to return a scalar value (typically, validation accuracy), which will be maximized by the search algorithm.

.. literalinclude:: dh_project/dh_project/advanced_hpo/model_run.py
    :linenos:
    :caption: advanced_hpo/model_run.py

Create ``problem.py`` to define the search space of hyperparameters for each model:

.. literalinclude:: dh_project/dh_project/advanced_hpo/problem.py
    :linenos:
    :caption: advanced_hpo/problem.py

Run the ``problem.py`` with ``python problem.py`` in your shell. The output will be:

.. code-block:: python
    :caption: [Out]

    Configuration space object:
    Hyperparameters:
        classifier, Type: Categorical, Choices: {RandomForest, GradientBoosting}, Default: RandomForest
        criterion, Type: Categorical, Choices: {friedman_mse, mse, mae, gini, entropy}, Default: gini
        learning_rate, Type: UniformFloat, Range: [0.01, 1.0], Default: 0.505
        loss, Type: Categorical, Choices: {deviance, exponential}, Default: deviance
        max_depth, Type: UniformInteger, Range: [1, 50], Default: 26
        min_samples_leaf, Type: UniformInteger, Range: [1, 10], Default: 6
        min_samples_split, Type: UniformInteger, Range: [2, 10], Default: 6
        n_estimators, Type: UniformInteger, Range: [1, 1000], Default: 32, on log-scale
        subsample, Type: UniformFloat, Range: [0.01, 1.0], Default: 0.505
    Conditions:
        learning_rate | classifier == 'GradientBoosting'
        loss | classifier == 'GradientBoosting'
        subsample | classifier == 'GradientBoosting'
    Forbidden Clauses:
        (Forbidden: classifier == 'RandomForest' && Forbidden: criterion in {'friedman_mse', 'mae', 'mse'})
        (Forbidden: classifier == 'GradientBoosting' && Forbidden: criterion in {'entropy', 'gini'})

Run the search for 20 model evaluations using the following command line:

.. code-block:: console
    :caption: bash

    deephyper hps ambs --problem dh_project.advanced_hpo.problem.Problem --run dh_project.advanced_hpo.model_run.run --max-evals 20 --evaluator ray --n-jobs 4

Once the search is over, the ``results.csv`` file contains the hyperparameters configurations evaluated during the search and their corresponding objective value (validation accuracy).
Create ``best_config.py`` as given below. It will extract the best configuration from the ``results.csv`` and run it for the training, validation and test set.

.. literalinclude:: dh_project/dh_project/advanced_hpo/best_config.py
    :linenos:
    :caption: advanced_hpo/best_config.py

Compared to the default configuration, we can see the accuracy improvement in the validation and test data sets.

.. code-block:: python
    :caption: [Out]

    Accuracy on Training: 0.754
    Accuracy on Validation: 0.664
    Accuracy on Testing: 0.664