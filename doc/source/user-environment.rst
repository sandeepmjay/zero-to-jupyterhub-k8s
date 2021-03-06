.. _user-environment:

Customizing User Environment
============================

This page contains instructions for common ways to enhance the user experience.
For a list of all the configurable Helm chart options, see the
:ref:`helm-chart-configuration-reference`.

The *user environment* is the set of software packages, environment variables,
and various files that are present when the user logs into JupyterHub. The user
may also see different tools that provide interfaces to perform specialized
tasks, such as JupyterLab, RStudio, RISE and others.

A :term:`Docker image` built from a :term:`Dockerfile` will lay the foundation for
the environment that you will provide for the users. The image will for example
determine what Linux software (curl, vim ...), programming languages (Julia,
Python, R, ...) and development environments (JupyterLab, RStudio, ...) are made
available for use.

To get started customizing the user environment, see the topics below.



.. _existing-docker-image:

Choose and use an existing Docker image
---------------------------------------

Project Jupyter maintains the `jupyter/docker-stacks repository
<https://github.com/jupyter/docker-stacks/>`_, which contains ready to use
Docker images. Each image includes a set of commonly used science and data
science libraries and tools. They also provide excellent documentation on `how
to choose a suitable image
<http://jupyter-docker-stacks.readthedocs.io/en/latest/using/selecting.html>`_.


If you wish to use another image from jupyter/docker-stacks than the
`base-notebook
<http://jupyter-docker-stacks.readthedocs.io/en/latest/using/selecting.html#jupyter-base-notebook>`_
used by default, such as the `datascience-notebook
<http://jupyter-docker-stacks.readthedocs.io/en/latest/using/selecting.html#jupyter-datascience-notebook>`_
image containing useful tools and libraries for datascience, complete these steps:

1. Modify your ``config.yaml`` file to specify the image. For example:

   .. code-block:: yaml

      singleuser:
        image:
          # Get the latest image tag at:
          # https://hub.docker.com/r/jupyter/datascience-notebook/tags/
          # Inspect the Dockerfile at:
          # https://github.com/jupyter/docker-stacks/tree/master/datascience-notebook/Dockerfile
          name: jupyter/datascience-notebook
          tag: 177037d09156

   .. note::

      Container image names cannot be longer than 63 characters.

      Always use an explicit ``tag``, such as a specific commit. Avoid using
      ``latest`` as it might cause a several minute delay, confusion, or
      failures for users when a new version of the image is released.

2. Apply the changes by following the directions listed in
   `apply the changes`_.
   
   
   .. note::
   
      If you have configured *prePuller.hook.enabled*, all the nodes in your
      cluster will pull the image before the the hub is upgraded to let users
      use the image. The image pulling may take several minutes to complete,
      depending on the size of the image.



.. _jupyterlab-by-default:

Use JupyterLab by default
-------------------------

`JupyterLab <http://jupyterlab.readthedocs.io/en/stable/index.html>`_ is a new
user interface for Jupyter about to replace the classic user interface (UI).
While users already can interchange ``/tree`` and ``/lab`` in the URL to switch between
the classic UI and JupyterLab, they will default to use the classic UI.

To let users use JupyterLab by default, add the following entries to your
:term:`config.yaml`:

.. code-block:: yaml

   singleuser:
     defaultUrl: "/lab"

   hub:
     extraConfig: |-
       c.Spawner.cmd = ['jupyter-labhub']

.. note::

   All images in the `jupyter/docker-stacks repository
   <https://github.com/jupyter/docker-stacks/>`_ come pre-installed with
   JupyterLab and the `JupyterLab-Hub extension
   <https://github.com/jupyterhub/jupyterlab-hub>`_ required for this
   configuration to work.



.. _custom-docker-image:

Customize an existing Docker image
----------------------------------

If you are missing something in the image that you would like all users to have,
we recommend that you build a new image on top of an existing Docker image from
jupyter/docker-stacks.

Below is an example :term:`Dockerfile` building on top of the *minimal-notebook*
image. This file can be built to a :term:`docker image`, and pushed to a
:term:`image registry`, and finally configured in :term:`config.yaml` to be used
by the Helm chart.

.. code-block:: Dockerfile

   FROM jupyter/minimal-notebook:177037d09156
   # Get the latest image tag at:
   # https://hub.docker.com/r/jupyter/minimal-notebook/tags/
   # Inspect the Dockerfile at:
   # https://github.com/jupyter/docker-stacks/tree/master/minimal-notebook/Dockerfile

   # install additional package...
   RUN pip install --yes astropy

.. note:

   If you are using a private image registry, you may need to setup the image
   credentials. See the :ref:`helm-chart-configuration-reference` for more
   details on this.



.. _set-env-vars:

Set environment variables
-------------------------

One way to affect your user's environment is by setting :term:`environment
variables`. While you can set them up in your Docker image if you build it
yourself, it is often easier to configure your Helm chart through values
provided in your :term:`config.yaml`.

To set this up, edit your :term:`config.yaml` and `apply the changes`_. For
example, this code snippet will set the environment variable ``EDITOR`` to the
value ``vim``:

.. code-block:: yaml

   singleuser:
     extraEnv:
       EDITOR: "vim"

You can set any number of static environment variables in the
:term:`config.yaml` file.

Users can read the environment variables in their code in various ways. In
Python, for example, the following code reads an environment variable's value:

.. code-block:: python

   import os
   my_value = os.environ["MY_ENVIRONMENT_VARIABLE"]



.. _add-files-to-home:

About user storage and adding files to it 
-----------------------------------------

It is important to understand the basics of how user storage is set up. By
default, each user will get 10GB of space on a harddrive that will persist in
between restarts of their server. This harddrive will be mounted to their home
directory. In practice this means that everything a user writes to the home
directory (`/home/jovyan`) will remain, and everything else will be reset in
between server restarts.

A server can be shut down by *culling*. By default, JupyterHub's culling service
is configured to cull a users server that has been inactive for one hour. Note
that JupyterLab will autosave files, and as long as the file was within the
users home directory no work is lost.




.. note::

   In Kubernetes, a *PersistantVolume* (PV) represents the harddrive.
   KubeSpawner will create a PersistantVolumeClaim that requests a PV from the
   cloud. By default, deleting the PVC will cause the cloud to delete the PV.

Docker image's $HOME directory will be hidden from the user. To make these
contents visible to the user, you must pre-populate the user's filesystem. To do
so, you would include commands in the ``config.yaml`` that would be run each
time a user starts their server. The following pattern can be used in
:term:`config.yaml`:

.. code-block:: yaml

   singleuser:
     lifecycleHooks:
       postStart:
         exec:
           command: ["cp", "-a", "src", "target"]

Each element of the command needs to be a separate item in the list. Note that
this command will be run from the ``$HOME`` location of the user's running
container, meaning that commands that place files relative to ``./`` will result
in users seeing those files in their home directory. You can use commands like
``wget`` to place files where you like.

However, keep in mind that this command will be run **each time** a user starts
their server. For this reason, we recommend using ``nbgitpuller`` to synchronize
your user folders with a git repository.



.. use-nbgitpuller:

Using ``nbgitpuller`` to synchronize a folder
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

We recommend using the tool `nbgitpuller
<https://github.com/jupyterhub/nbgitpuller>`_ to synchronize a folder
in your user's filesystem with a ``git`` repository whenever a user
starts their server.  This synchronization can also be triggered by
letting a user visit a link like
``https://your-domain.com/hub/user-redirect/git-pull?repo=https://github.com/data-8/materials-fa18``
(e.g., as alternative start url).

To use ``nbgitpuller``, first make sure that you `install it in your Docker
image <https://github.com/jupyterhub/nbgitpuller#installation>`_. Once this is done,
you'll have access to the ``nbgitpuller`` CLI from within JupyterHub. You can
run it with a ``postStart`` hook with the following configuration

.. code-block:: yaml

   singleuser:
     lifecycleHooks:
       postStart:
         exec:
           command: ["gitpuller", "https://github.com/data-8/materials-fa17", "master", "materials-fa"]

This will synchronize the master branch of the repository to a folder called
``$HOME/materials-fa`` each time a user logs in. See `the nbgitpuller
documentation <https://github.com/jupyterhub/nbgitpuller>`_ for more information on
using this tool.

.. warning::

   ``nbgitpuller`` will attempt to automatically resolve merge conflicts if your
   user's repository has changed since the last sync. You should familiarize
   yourself with the `nbgitpuller merging behavior
   <https://github.com/jupyterhub/nbgitpuller#merging-behavior>`_ prior to using the
   tool in production.



.. setup-conda-envs:

Allow users to create their own ``conda`` environments
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Sometimes you want users to be able to create their own ``conda`` environments.
By default, any environments created in a JupyterHub session will not persist
across sessions. To resolve this, take the following steps:

1. Ensure the ``nb_conda_kernels`` package is installed in the root
   environment (e.g., see :ref:`r2d-custom-image`)

2. Configure Anaconda to install user environments to a folder within ``$HOME``.

   Create a file called ``.condarc`` in the home folder for all users, and make
   sure that the following lines are inside:

   .. code-block:: yaml

      envs_dirs:
        - /home/jovyan/my-conda-envs/

  The text above will cause Anaconda to install new environments to this folder,
  which will persist across sessions.



.. REFERENCES USED:

.. _apply the changes: extending-jupyterhub.html#apply-config-changes
.. _downloading and installing Docker: https://www.docker.com/community-edition
.. _pip: https://pip.readthedocs.io/en/latest/user_guide/#requirements-files
