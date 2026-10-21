.. _infra-ci-gha-workflows:

GitHub Actions workflows
========================

Every CCCL workflow lives in ``.github/workflows/``. They fall into five purposes:
the CI matrix, manual developer tools, the release pipeline, documentation
deployment, and repository automation. This page catalogs each workflow, then
explains the workflow-file mechanics that turn one push into hundreds of jobs.
For the high-level CI narrative — triggering, change detection, the ``ci:`` gate —
see :ref:`infra-ci-overview`.

Reusable workflows
------------------

``ci-workflow-pull-request.yml``, ``ci-workflow-nightly.yml``, and
``ci-workflow-weekly.yml`` support ``workflow_call`` alongside their push,
schedule, or manual triggers.
``ci-workflow-custom.yml`` provides manual dispatch of ``ci-matrix-build.yml``,
passing its ``workflows`` and ``matrix_ref`` inputs. The ``workflows`` input
defaults to ``nightly``. Other workflows call ``ci-matrix-build.yml`` directly.
Nested workflow calls use ``$/.github/workflows/...``
to reference implementations in the workflow's own repository and commit.

.. note::
   The use of ``$/`` references require self-hosted runners to use
   GitHub Actions runner version 2.336.0 or newer.

Callers must grant the token permissions needed by the called jobs.
Pass any required secrets through an explicit secret mapping, or use
``secrets: inherit`` within the same organization or enterprise. CI profiles
control concurrency for their runs.

Matrix selection
~~~~~~~~~~~~~~~~

Callable CI profiles and the matrix builder test the triggering commit in the
caller repository. Matrix files also come from the caller repository, but their
ref can be selected independently and defaults to the calling ref.

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Input
     - Purpose
   * - ``matrix_ref``
     - Matrix branch, tag, or commit SHA.
   * - ``matrix_path``
     - Matrix file path relative to the repository root. Defaults to
       ``ci/matrix.yaml``.

Matrix-driven CI generates jobs from the selected matrix file and runs them
against the triggering commit.

For example, a caller can select a matrix from another ref:

.. code-block:: yaml

    jobs:
      ci:
        uses: NVIDIA/cccl/.github/workflows/ci-workflow-nightly.yml@main
        permissions:
          contents: read
          actions: read
          id-token: write
        with:
          matrix_ref: ci-config
          matrix_path: ci/matrix.yaml

Source and matrix access, run artifacts, and pull request comments use the
caller's ``GITHUB_TOKEN``.

Matrix jobs fall back to GitHub-hosted runners for forks. Repositories enabled
for self-hosted CI use the matrix's runner labels. The PR profile expects a
``pull-request/<N>`` branch and requires ``pull-requests: write`` to post comments.

Workflow reference
------------------

CI matrix
~~~~~~~~~

The CI profiles select a matrix key and run it. Six reusable dispatcher workflows
unfold the generated job list into GitHub matrix strategies. Contributors rarely
invoke any of these directly; they fire on push or schedule.

.. list-table::
   :header-rows: 1
   :widths: 40 20 40

   * - Workflow
     - Trigger
     - Purpose
   * - ``.github/workflows/ci-workflow-pull-request.yml``
     - Automatic / reusable
     - Runs the ``pull_request`` matrix on every push to a ``pull-request/<N>`` branch.
   * - ``.github/workflows/ci-workflow-nightly.yml``
     - Scheduled / manual / reusable
     - Runs the broader ``nightly`` matrix on a weekday schedule, with AI failure analysis and Slack notifications.
   * - ``.github/workflows/ci-workflow-weekly.yml``
     - Scheduled / manual / reusable
     - Runs the exhaustive ``weekly`` matrix on a Sunday schedule, with AI failure analysis and Slack notifications.
   * - ``.github/workflows/ci-workflow-custom.yml``
     - Manual
     - Runs selected workflows from a matrix on another branch in the same
       repository; defaults to ``nightly``.
   * - ``.github/workflows/ci-matrix-build.yml``
     - Reusable
     - Builds and dispatches the selected matrix and summarizes its results.
   * - ``.github/workflows/workflow-dispatch-standalone-group-linux.yml``
     - Automatic
     - Dispatches an array of standalone Linux jobs as a matrix.
   * - ``.github/workflows/workflow-dispatch-standalone-group-windows.yml``
     - Automatic
     - Dispatches an array of standalone Windows jobs as a matrix.
   * - ``.github/workflows/workflow-dispatch-two-stage-group-linux.yml``
     - Automatic
     - Dispatches an array of producer/consumer Linux chains as a matrix.
   * - ``.github/workflows/workflow-dispatch-two-stage-group-windows.yml``
     - Automatic
     - Dispatches an array of producer/consumer Windows chains as a matrix.
   * - ``.github/workflows/workflow-dispatch-two-stage-linux.yml``
     - Automatic
     - Executes one producer/consumer chain on Linux.
   * - ``.github/workflows/workflow-dispatch-two-stage-windows.yml``
     - Automatic
     - Executes one producer/consumer chain on Windows.

Manual tools
~~~~~~~~~~~~

Contributors invoke these through ``workflow_dispatch`` inputs on the Actions tab.

.. list-table::
   :header-rows: 1
   :widths: 40 20 40

   * - Workflow
     - Trigger
     - Purpose
   * - ``.github/workflows/bench.yml``
     - Both
     - Compares benchmark performance between two refs; also called by the PR workflow's bench dispatch.
   * - ``.github/workflows/git-bisect.yml``
     - Manual
     - Runs an automated ``git bisect`` across a commit range to locate a regression.

Documentation
~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 40 20 40

   * - Workflow
     - Trigger
     - Purpose
   * - ``.github/workflows/docs-deploy.yml``
     - Both
     - Builds the Sphinx docs and publishes them to GitHub Pages under a version-derived path.


CI orchestration
----------------

The build-workflow action
~~~~~~~~~~~~~~~~~~~~~~~~~

The "build workflow from matrix" job runs the ``build-workflow`` action at
``.github/actions/workflow-build/``. Its script ``build-workflow.py`` reads
the selected matrix file together with any change-detection output from
``ci/inspect_changes.py`` and produces ``workflow/workflow.json``. That file holds
named groups — "CUB GCC", for example — each carrying a ``standalone`` array and a
``two_stage`` array of fully expanded job specs.

A second script, ``prepare-workflow-dispatch.py``, splits ``workflow.json`` into
``dispatch.json``. This is the structure the GHA workflow is actually dispatched over.

The four-bucket split
~~~~~~~~~~~~~~~~~~~~~

``dispatch.json`` carries four buckets, keyed by operating system crossed with job
structure:

.. code-block:: text

    linux_standalone     linux_two_stage
    windows_standalone   windows_two_stage

Each bucket holds an ordered ``keys`` list of group names and a ``jobs`` map from
each key to its job specs:

.. code-block:: text

    {
      "linux_two_stage": {
        "keys": ["CUB GCC", "Thrust Clang", ...],
        "jobs": {
          "CUB GCC":      [{ "producers": [...], "consumers": [...] }],
          "Thrust Clang": [{ "producers": [...], "consumers": [...] }]
        }
      },
      "linux_standalone":   { "keys": [...], "jobs": {...} },
      "windows_two_stage":  { "keys": [...], "jobs": {...} },
      "windows_standalone": { "keys": [...], "jobs": {...} }
    }

``ci-workflow-pull-request.yml`` dispatches four parallel matrix jobs, one per
bucket. Each fans out over its bucket's ``keys`` and calls the matching dispatcher
workflow — ``workflow-dispatch-<structure>-group-<os>.yml``.

Standalone jobs
~~~~~~~~~~~~~~~

A standalone job builds and tests on one runner, exchanging no artifacts with other
jobs. The group dispatcher
(``workflow-dispatch-standalone-group-<os>.yml``) fans out over the bucket's job
array; each matrix entry runs one CI script — ``ci/build_<project>.sh`` or
``ci/test_<project>.sh`` — start to finish. See :ref:`infra-ci-scripts` for details.

Two-stage jobs
~~~~~~~~~~~~~~

A typical two-stage workflow splits a build job from one or more test jobs. Each stage
has different workloads and hardware requirements, and this division allows us to schedule
each on appropriate hardware.

The producer runs on a GPU-less, CPU-heavy runner.
It builds and uploads binaries through GHA artifacts (:ref:`infra-ci-artifacts`) or sccache.

Each consumer runs on a CPU-light, GPU-attached runner.
They fetch the pre-compiled test binaries and execute them.
