.. _organizing-models:

#######################################
 Organize Models in the Model Registry
#######################################

Determined includes built-in support for a *model registry*, which makes it easy to organize trained
models and their respective versions. Common use-cases for the model registry include:

-  Grouping related checkpoints together, including checkpoints across experiments.

-  Storing metadata about a model that is specific to your problem or organization. Examples include
   references to production systems, dataset links, Git links, and metrics calculated outside of
   Determined.

-  Retrieving the latest version of a model for downstream tasks like serving or batch inference.

The model registry contains a set of *models*. Each model has a unique name and zero or more *model
versions*. A model version consists of a version number and a checkpoint, which represents the state
of a trained model.

The model registry is designed to be flexible, and the best way to use it depends on your
organization's requirements and workflow. For example, one approach is to define a model for each
high-level task you want to use machine learning for (e.g., "object-detection",
"sentiment-analysis", etc.). Then each version of this model would correspond to a new approach to
solving that task. Note that different versions of a model might come from different experiments,
use different network architectures, or even use different deep learning frameworks. Another
approach would be to register a model named "FasterRCNN", and ensure that each version of the model
uses that network architecture.

***************
 Manage Models
***************

A *model* has a unique name, an optional description, user-defined metadata, and zero or more *model
versions*. A model's metadata can contain arbitrary information about the model. The following is an
example JSON representation of a model for illustration.

.. code:: json

   {
     "mnist_cnn": {
       "description": "a character recognition model",
       "metadata": {
         "dataset_url": "http://yann.lecun.com/exdb/mnist/",
         "git_url": "http://github.com/user/repo"
       },
       "versions": []
     }
   }

Use the WebUI
=============

Models can be created and edited through the WebUI. Some features can only be accessed through the
WebUI, such as writing longform notes in Markdown.

A new model can be created on the Model Registry page, which can be found in the navigation bar.
This launches a modal where basic information can be specified before creation. A new version can be
added to a model from the Checkpoint modal. This launches a model where basic information can be
specified before registering the model version. A new model can also be created from here.

Once a model and/or version is created, notes, metadata, and other information can be added and
edited through the Model Registry page after selecting the model or model version in question.

Register Models
===============

A model can be added to the registry via the WebUI, Python API, REST API, or CLI. This guide will
cover the Python and CLI methods. For information on the REST API, see the `Swagger API
documentation <../rest-api/index.html#/Models>`__.

The following example demonstrates how to add a new model to the registry;
:func:`~determined.experimental.Determined.create_model` returns an instance of the
:class:`~determined.experimental.Model` class. The new model will not have any versions (model
checkpoints) associated with it; adding versions to a model is described below.

.. code:: python

   from determined.experimental import Determined

   model = Determined().create_model(
       "model_name",
       description="optional description",
       metadata={"optional": "JSON serializable dictionary"},
   )

Similarly, you can create a model from the CLI using the following command.

.. code:: bash

   det model create <model_name>

Query Models
============

The following example returns models registered in Determined as a list of
:class:`~determined.experimental.Model` objects. Models can be sorted by name, description, creation
time, and last updated time. Additionally, models can be filtered by name or description via the
Python API. For sorting and ordering options, see :class:`~determined.experimental.ModelSortBy` and
:class:`~determined.experimental.ModelOrderBy` respectively.

.. code:: python

   from determined.experimental import Determined, ModelOrderBy

   d = Determined()

   all_models = d.get_models()

   chronological_sort = d.get_models(sort_by=ModelSortBy.CREATION_TIME)

   # Find all models with "mnist" in their name. Some possible model names
   # are "mnist_pytorch", "mnist_cnn", "mnist", etc.
   mnist_models = d.get_models(name="mnist")

   # Find all models whose description contains "ocr".
   ocr_models = d.get_models(description="ocr")

Similarly, you can list models from the CLI using the following command.

.. code:: bash

   det model list --sort-by={name,description,creation_time,last_updated_time} --order-by={asc,desc}

The following snippet queries for a single model by name.

.. code:: python

   from determined.experimental import Determined

   model = Determined().get_model("model_name")

The CLI equivalent is below. The ``describe`` command will print information about the latest
version of the model by default as well.

.. code:: bash

   det model describe <model_name>

Modify Model Metadata
=====================

Currently, model metadata can only be edited via the WebUI and Python API. The following example
demonstrates how to use this API.

.. code:: python

   from determined.experimental import Determined

   model = Determined().get_model("model_name")

   # Metadata is merged with existing metadata.
   model.add_metadata({"key", "value"})
   model.add_metadata({"metrics": {"test_set_loss": 0.091}})

   # Result: {"key": "value", "metrics": {"test_set_loss": 0.091}}.

   # Only top-level keys are merged. The following statement will replace the
   # previous value of the "metrics" key.
   model.add_metadata({"metrics": {"test_set_acc": 0.97}})

   # Result: {"key": "value", "metrics": {"test_set_acc": 0.97}}.

   model.remove_metadata(["key"])

   # Result: {"metrics": {"test_set_acc": 0.97}}.

***********************
 Manage Model Versions
***********************

Once a model has been added to the registry, you can add one or more checkpoints to it. These
registered checkpoints are known as *model versions*. Version numbers are assigned by the registry;
version numbers start at ``1`` and increment each time a new model version is registered.

For illustration, this JSON document illustrates an example model with a single registered version.

.. code:: json

   {
     "mnist_cnn": {
       "description": "a character recognition model",
       "metadata": {
         "dataset_url": "http://yann.lecun.com/exdb/mnist/",
         "git_url": "http://github.com/user/repo"
       },
       "versions": [
         {
           "version_number": 1,
           "checkpoint": {
             "uuid": "6a24d772-f1f7-4655-9061-22d582afd96c",
             "experiment_config": { "...": "..." },
             "experimentId": 1,
             "trialId": 1,
             "hparams": { "...": "..." },
             "batchNumber": 100,
             "resources": { "...": "..." },
             "metadata": {},
             "framework": "tensorflow-1.14.0",
             "format": "h5",
             "metrics": { "...": "..." }
           }
         }
       ]
     }
   }

Create Versions
===============

The following snippet registers a new version of a model.
:func:`~determined.experimental.Model.register_version()` returns an updated
:class:`~determined.experimental.Checkpoint` object representing the new model version.

.. code:: python

   from determined.experimental import Determined

   d = Determined()

   checkpoint = d.get_experiment(exp_id).top_checkpoint()

   model = d.get_model("model_name")

   model_version = model.register_version(checkpoint.uuid)

Similarly, a new model version can be registered using the CLI as follows:

.. code:: bash

   det model register-version <model_name> <checkpoint_uuid>

Access Versions
===============

The example below demonstrates how to retrieve versions of a model from the registry. If no version
number is specified, the most recent version of the model is returned.
:func:`~determined.experimental.Model.get_version()` returns an instance of
:class:`~determined.experimental.Checkpoint`; as shown in the example, this makes it easy to perform
common operations like downloading the checkpoint to local storage or loading the trained model into
memory.

.. code:: python

   from determined.experimental import Determined

   model = Determined().get_model("model_name")

   specific_version = model.get_version(3)
   latest_version = model.get_version()

   # Depending on the framework used to create the checkpoint, loading from
   # the checkpoint may return either a PyTorchTrial instance or a TensorFlow
   # object representing the trained model.
   path = latest_version.checkpoint.download()

   from determined import pytorch
   my_pytorch_trial = pytorch.load_trial_from_checkpoint_path(path)

   from determined import keras
   my_keras_model = keras.load_model_from_checkpoint_path(path)

The following example lists all the versions of a model. By default, model versions are returned in
descending order such that the most recent versions are returned first.

.. code:: python

   from determined.experimental import Determined

   model = Determined().get_model("model_name")

   model_versions = model.get_versions()

The CLI equivalent is as follows:

.. code:: bash

   det model list-versions <model_name>

************
 Next Steps
************

-  :ref:`python-api-reference`: The reference documentation for this API.
-  :ref:`use-trained-models`
