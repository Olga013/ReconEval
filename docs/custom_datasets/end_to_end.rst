.. _custom-data-end-to-end:

End-to-end reconstruction
-------------------------

:doc:`Back to introduction <../custom_datasets>`

For a worked example, see the
:doc:`End-to-end tutorial <../tutorials/end_to_end>`.

1. Prepare inputs
~~~~~~~~~~~~~~~~~

Provide data for the chosen workflow:

- For training:
  
  - expression per split (train, val, test)

- For evaluation and embedding extraction:

  - a grouped expression store
  - split metadata (evaluation only)

The preprocessing example on the LuCA dataset: ``experiments/preprocessing/preprocess_luca.py``.

.. _custom-data-e2e-training-inputs:

1.1. Training
^^^^^^^^^^^^^

- For AE and scVI variants, provide one cells-by-genes matrix per
  split, stored as a Zarr array or under ``X``:

  .. code-block:: text

     /path/to/mydata/
     ├── train.zarr
     ├── val.zarr
     └── test.zarr

- PCA instead reads ``train.h5ad`` from the same directory, with expression
  stored as a CSR matrix in ``.X``

.. _custom-data-e2e-reference-pca-inputs:

1.2. Reference PCA
^^^^^^^^^^^^^^^^^^

Combine measured training, validation, and test expression in
``/path/to/mydata/reference.h5ad``, including each cell once.
Store a cells-by-genes CSR matrix in ``.X``, preserving evaluation
preprocessing and gene order.

Use this file only to fit the reference PCA.

.. _custom-data-e2e-eval-inputs:

1.3. Evaluation
^^^^^^^^^^^^^^^

In order to run evaluation, provide following data:

- a grouped expression store
- split metadata

Place both under your dataset directory:

.. code-block:: text

  /path/to/mydata/
  ├── expression.zarr/
  │   ├── <group>/
  │   │   └── X          # cells × genes
  │   │   └── obs_index  # optional cell IDs
  │   └── ...
  ├── split_metadata.zarr/
  │   ├── .zgroup        # Zarr format information
  │   └── .zattrs        # train/val/test group-name lists
  └── ...                # train.zarr, val.zarr, test.zarr (Optional)   


.. dropdown:: Note: Held-out evaluation parameters
   :icon: info
   :color: info
   :animate: fade-in-slide-down

   For held-out evaluation set ``train_fraction=0`` and ``test_fraction=1`` in configuration 
   to evaluate only groups listed in ``test_combinations``.

.. dropdown:: Note: Evaluation expression data
   :icon: info
   :color: info
   :animate: fade-in-slide-down

   Expression is loaded from ``expression.zarr/<group>/X``; and 
   ``train.zarr``, ``val.zarr``, ``test.zarr`` are not used for evaluation.

**Expression.**

Group cells by ``data.split_key`` joining the values with the ``-`` symbol to name each group:


.. csv-table::
   :header: "Dataset", "``data.split_key``", "Example group name"
   :widths: 15, 45, 40

   "LuCA", "``[cell_type, dataset, origin]``", "``T_cell-study1-tumor``"
   "PBMC", "``[cell_type, donor, cytokine]``", "``T_cell-donor1-PBS``"
   "Tahoe", "``[cell_line, drug, dosage]``", "``cell_line1-drug_A-1``"

Expression store includes:

- ``<group>/X``: cells-by-genes expression
- ``<group>/obs_index``: optional cell IDs in matching row order
- Root ``var_names`` attribute: ordered gene names for biological evaluation

**Split metadata.**
Create ``split_metadata.zarr`` using the Zarr library. It should contain:

- ``.zattrs``: train/val/test group-name lists:

  - ``train_combinations``
  - ``val_combinations``
  - ``test_combinations``

- ``.zgroup``: Zarr format information

Example ``.zattrs`` contents:

.. code-block:: json

   {
     "train_combinations": ["T_cell-study1-tumor"],
     "val_combinations": ["T_cell-study2-tumor"],
     "test_combinations": ["T_cell-study3-tumor"]
   }

Example ``.zgroup`` contents (Zarr format version 2):

.. code-block:: json

   {
     "zarr_format": 2
   }


For different split definitions, reuse the expression
store and recreate separate split metadata.

.. _custom-data-e2e-embedding-inputs:

1.4. Embedding extraction
^^^^^^^^^^^^^^^^^^^^^^^^^
Use your trained model to obtain embeddings from ``expression.zarr`` prepared in :ref:`section 1.3 <custom-data-e2e-eval-inputs>`.
The extractor processes all groups in the store; split metadata is not required.

.. _custom-data-e2e-data-config:

1.5. Data configuration
^^^^^^^^^^^^^^^^^^^^^^^

After preparing the files, copy
``experiments/01_end_to_end/configs/data/template.yaml`` to
``experiments/01_end_to_end/configs/data/mydata.yaml``. Set these top-level
fields and retain the ``model_specific`` dataloader settings:

.. code-block:: yaml

   name: mydata
   path: /path/to/mydata  # directory containing the split files
   input_dim: 2000        # number of genes

   # Set these fields for evaluation
   eval_path: /path/to/mydata/expression.zarr
   split_comb: /path/to/mydata/split_metadata.zarr
   split_key: 
     - cell_type
     - dataset
     - origin

``split_key`` uses the LuCA example above; replace it with the fields
and order used to name your groups.

Select this configuration with ``data=mydata`` when running the scripts.

2. Train
~~~~~~~~

2.1. Configure
^^^^^^^^^^^^^^

Choose a model:

- **AE variants** with three library-size options:

  - **AE:** no library-size scaling, default
  - **olAE:** AE with observed library size
  - **mlAE:** AE with learned library size

- **VAE variants**:

  - **scVI**
  - **nlscVI**
  - **mlscVI**

- **PCA**
- or **your own model** (see
  `Bringing your own model <../tutorials/end_to_end.html#bringing-your-own-model>`_)

Reuse the dataset configuration prepared in
:ref:`section 1.5 <custom-data-e2e-data-config>`.

Set the parameters in YAML or pass them as ``key=value`` on the CLI:

.. dropdown:: Training parameters

   .. tab-set::

      .. tab-item:: Common settings

         .. csv-table::
            :header: "Parameter", "Required / optional", "Value / meaning"
            :widths: 40, 25, 35

            "``data``", "Required", "Dataset configuration, e.g. ``mydata``"
            "``model``", "Required", "Training configuration: ``train/<model>``; use ``train/AE`` for all AE variants"
            "``trainer``", "Required", "``AE`` for AE variants; ``scVI`` for all VAE variants; ``PCA`` for PCA"
            "``data.name``", "Required", "Dataset label used in output paths and logs"
            "``data.path``", "Required", "Directory containing the training, validation, and test files"
            "``data.input_dim``", "Required", "Number of expression columns (genes)"
            "``data.model_specific.<name>.minibatch_size``", "Optional; default: ``256``", "Cells per batch for AE and VAE variants; replace ``<name>`` with the value of ``model.meta.name``"
            "``data.model_specific.<name>.num_workers``", "Optional; default: ``0``", "Data-loading workers for AE and VAE variants; replace ``<name>`` with the value of ``model.meta.name``"
            "``trainer.max_epochs``", "Optional; default: ``100``", "Maximum training epochs for AE and VAE variants"
            "``trainer.min_epochs``", "Optional; default: AE ``10``; VAE ``30``", "Minimum training epochs"
            "``seed``", "Optional; default: ``42``", "Random seed"
            "``split``", "Optional; default: ``split03``", "Split label used in output paths"

      .. tab-item:: AE variants

         .. csv-table::
            :header: "Parameter", "Required / optional", "Value / meaning"
            :widths: 40, 25, 35

            "``model.meta.name``", "Required for olAE/mlAE; default: ``AE``", "``AE``, ``olAE``, or ``mlAE``; selects the dataloader settings and labels model outputs"
            "``model.model_args.library_size_mode``", "Required for olAE/mlAE; default: ``none``", "``none``: **AE** - no library-size scaling, default; ``observed``: **olAE** - AE with observed library size; ``modeled``: **mlAE** - AE with learned library size"
            "``model.model_args.n_hidden``", "Optional; default: ``[1024]``", "Hidden-layer widths; list length determines depth"
            "``model.model_args.n_latent``", "Optional; default: ``100``", "Latent dimension"

      .. tab-item:: VAE variants

         .. csv-table::
            :header: "Parameter", "Required / optional", "Value / meaning"
            :widths: 40, 25, 35

            "``model.model_args.n_hidden``", "Optional; default: ``1024``", "Hidden-layer width"
            "``model.model_args.n_latent``", "Optional; default: ``300``", "Latent dimension"
            "``model.model_args.n_layers``", "Optional; default: ``3``", "Number of hidden layers"
            "``model.model_args.use_observed_lib_size``", "Optional; scVI/nlscVI default: ``true``; mlscVI default: ``false``", "``true`` uses observed totals; ``false`` infers library size"

      .. tab-item:: PCA

         .. csv-table::
            :header: "Parameter", "Required / optional", "Value / meaning"
            :widths: 40, 25, 35

            "``model.model_args.n_components``", "Optional; default: ``300``", "Number of principal components to fit"

Defaults above refer to the provided data template and training configurations.

.. _custom-data-e2e-pca-setup:

.. dropdown:: Note: hardcoded temporary directory
   :icon: info
   :color: info
   :animate: fade-in-slide-down

   Before running PCA, replace the default ``temp_dir`` in
   ``ReconPCA._setup_cluster``
   (``src/sc_reconstruction/models/reconpca.py``) with a writable
   directory for temporary files on your machine.

.. _custom-data-e2e-cli:

2.2. Run
^^^^^^^^

Activate ``reconeval`` using ``cstm_scvi_env.yaml`` from the
:ref:`installation step <custom-data-installation>` and run from the
repository root. For cluster submission, follow
:ref:`Run through SLURM <slurm-parameters>`.

**Run with a YAML configuration:**

For example, train an AE with your ``mydata.yaml`` configuration:

.. code-block:: bash

   python experiments/01_end_to_end/codes/train.py \
     model=train/AE \
     trainer=AE \
     data=mydata \
     model.model_args.n_latent=128

**Alternatively, you can use command-line overrides:**

Use the same copied ``mydata.yaml`` and supply or override its settings
for an individual run:

.. code-block:: bash

   python experiments/01_end_to_end/codes/train.py \
     model=train/AE \
     trainer=AE \
     data=mydata \
     data.name=new_mydata \
     data.path=/path/to/new_mydata

.. dropdown:: Note: CLI overrides
   :icon: info
   :color: info
   :animate: fade-in-slide-down

   - Both options use Hydra's ``key=value`` syntax
   - ``data=mydata`` selects ``configs/data/mydata.yaml``
   - Command-line values take precedence over YAML

2.3. Outputs
^^^^^^^^^^^^

.. _custom-data-e2e-model-files:

**Model files**

Training saves model outputs under:

- ``RECONEVAL_OUT`` is set: ``$RECONEVAL_OUT/weights/mydata/<split>/<model>/<note>/``
- ``RECONEVAL_OUT`` is unset: ``~/reconeval_outputs/weights/<dataset>/<split>/<model>/<note>/``

The directory names correspond to configuration settings:

- ``<dataset>``: ``data.name``, such as ``mydata``
- ``<split>``: ``split``, which defaults to ``split03``
- ``<model>``: ``model.meta.name``, such as ``AE`` or ``scVI``
- ``<note>``: ``model.meta.note``, which defaults to ``Default``

**Logs and configuration**

Using the same output root:

- ``logs/`` is the configured W&B logging directory
- ``outputs/<dataset>/<model>/<YYYY-MM-DD_HH-MM>/`` contains Hydra's
  run files, including configuration and overrides under ``.hydra/``

.. _custom-data-e2e-reference-pca:

3. Prepare reference PCA
~~~~~~~~~~~~~~~~~~~~~~~~

For distributional evaluation, reuse a matching reference PCA or fit one
using the :ref:`reference expression <custom-data-e2e-reference-pca-inputs>`
and :ref:`dataset configuration <custom-data-e2e-data-config>`.

After completing the :ref:`PCA setup <custom-data-e2e-pca-setup>`,
run from the repository root in GPU-enabled ``reconeval``, directly
or through :ref:`SLURM <slurm-parameters>`.

The example fits 50 components:

.. code-block:: bash

   python experiments/01_end_to_end/codes/train.py \
     data=mydata \
     model=train/PCA \
     trainer=PCA \
     data.model_specific.PCA.train_path=/path/to/mydata/reference.h5ad \
     model.model_args.n_components=50 \
     trainer.save_path=/path/to/reference_pca

Outputs in ``/path/to/reference_pca/``:

- ``mean.zarr``: mean expression per gene
- ``pc_50.zarr``: the genes-by-50 component matrix

Reuse these files across models with matching preprocessing and gene
order.

4. Evaluate
~~~~~~~~~~~
The evaluation consists of two steps:

- Reconstruct the held-out data
- Score the measured and reconstructed expression

To explore a toy example, see the :doc:`End-to-end tutorial <../tutorials/end_to_end>`.
This section explains how to configure and run the evaluation scripts from the Python CLI.

For end-to-end evaluation, the framework supports:

- Statistical metrics
- Biological metrics

For details, see `Metric families <../overview.html#metric-families>`_ and the
:doc:`Metrics tutorial <../tutorials/metrics>`.

.. _custom-data-e2e-eval-config:

4.1. Configure
^^^^^^^^^^^^^^

For the evaluation scripts, reuse the dataset configuration from
:ref:`section 1.5 <custom-data-e2e-data-config>`, select a
metric configuration under ``experiments/01_end_to_end/configs/metric/``,
and create a model evaluation YAML under ``experiments/01_end_to_end/configs/model/eval/``.

Example for AE (``model/eval/AE.yaml``):

.. code-block:: yaml

   meta:
     name: AE

   load:
     path: "/path/to/weights/mydata/<split>/AE/Default/<max_epochs>_<n_hidden>_<n_latent>_<YYYYMMDD>/epoch=<epoch>-val/loss_epoch=<val_loss>.ckpt"
     model_name: "<max_epochs>_<n_hidden>_<n_latent>_<YYYYMMDD>"
     additional_params:
       distribution: normal
       input_dim: ${data.input_dim}

Use the exact checkpoint path and the matching run-directory name; see :ref:`Model files <custom-data-e2e-model-files>`.

Set the parameters in YAML or pass them as ``key=value`` on the CLI:

.. dropdown:: Evaluation parameters

   .. tab-set::

      .. tab-item:: Common settings

         .. csv-table::
            :header: "Parameter", "Required / optional", "Value / meaning"
            :widths: 40, 25, 35

            "``data``", "Required", "Dataset configuration, e.g. ``mydata``"
            "``model``", "Required", "Evaluation configuration: ``eval/<model>``"
            "``metric``", "Required", "Selected metric configuration"
            "``model.load.path``", "Required", "Checkpoint path; for PCA, an ordered list ``[mean_path, components_path]``"
            "``model.load.model_name``", "Required", "Matching training-run identifier; for PCA, a label such as ``pc_128``. Used in the default output path"
            "``dynamic_model_loading``", "Optional; default: ``true``", "Keep ``true`` for configurations using ``load.additional_params``"
            "``train_fraction``", "Required for held-out-only evaluation", "Set to ``0`` to exclude training groups; current default: ``0.1``"
            "``test_fraction``", "Optional; default: ``1``", "Fraction of held-out groups to evaluate"
            "``seed``", "Optional; default: ``42``", "Group-sampling seed"
            "``split``", "Optional; default: ``split01``", "Match the trained split label; actual group assignments come from ``data.split_comb``"
            "``output.save_path``", "Required for custom output", "Output CSV path; use a separate path for each model and metric configuration"

      .. tab-item:: AE variants

         .. csv-table::
            :header: "Parameter", "Required / optional", "Value / meaning"
            :widths: 40, 25, 35

            "``model.load.additional_params.input_dim``", "Required for dynamic AE construction", "``${data.input_dim}``; match the training gene count"
            "``model.load.additional_params.distribution``", "Optional; default: ``normal``", "Match the distribution used during training"
            "``model.load.additional_params.library_size_mode``", "Optional; default: ``none``", "Match the trained variant: AE ``none``, olAE ``observed``, mlAE ``modeled``"

      .. tab-item:: VAE variants

         .. csv-table::
            :header: "Parameter", "Required / optional", "Value / meaning"
            :widths: 40, 25, 35

            "``model.load.additional_params.gene_likelihood``", "Optional; match training", "Likelihood used to train scVI, nlscVI, or mlscVI"
            "``model.load.additional_params.use_observed_lib_size``", "Optional; match training", "Current training defaults: scVI/nlscVI ``true``; mlscVI ``false``"

      .. tab-item:: PCA

         .. csv-table::
            :header: "Parameter", "Required / optional", "Value / meaning"
            :widths: 40, 25, 35

            "``model.model_args._target_``", "Required in the PCA YAML", "``sc_reconstruction.models.reconpca.ReconPCA``"

      .. tab-item:: Metric settings

         .. csv-table::
            :header: "Parameter", "Required / optional", "Value / meaning"
            :widths: 40, 25, 35

            "``metric.evaluator.mode``", "Required", "``reconstruction``"
            "``metric.evaluator.emb_zarr_path``", "Optional", "Omit or set to ``null`` for reconstruction"

      .. tab-item:: Reference PCA

         .. csv-table::
            :header: "Parameter", "Required / optional", "Value / meaning"
            :widths: 40, 25, 35

            "``metric.projector.load.path``", "Required for distributional evaluation", "Ordered paths to the saved reference mean and components; reuse the reference PCA across models with matching preprocessing and gene order"


.. dropdown:: Note: Reference PCA configuration
   :icon: info
   :color: info
   :animate: fade-in-slide-down

   For distributional evaluation, use the files from
   :ref:`Prepare reference PCA <custom-data-e2e-reference-pca>`.
   Create ``experiments/01_end_to_end/configs/metric/projector/PCA.yaml``
   with these paths:

   .. code-block:: yaml

      model_args:
        _target_: sc_reconstruction.models.reconpca.ReconPCA

      load:
        path:
          - /path/to/reference_pca/mean.zarr
          - /path/to/reference_pca/pc_50.zarr

4.2. Run
^^^^^^^^

Run from the repository root in the ``reconeval`` environment, or follow
:ref:`Run through SLURM <slurm-parameters>`.

Replace ``<model>`` with the name of the model evaluation YAML created in
:ref:`section 4.1 <custom-data-e2e-eval-config>`. The templates assume that ``data/mydata.yaml`` and
``model/eval/<model>.yaml`` are configured, including the checkpoint path
and matching training-run name. Both commands reconstruct and score the
groups listed in ``test_combinations``.

**Distributional metrics.**

Compute the following metrics:

- **On expression:** :func:`~sc_reconstruction.metrics.metric_r2`,
  :func:`~sc_reconstruction.metrics.metric_mse`
- **Using the reference PCA:** MMD (``mmd_rbf()``) and
  :func:`~sc_reconstruction.metrics.metric_energy_distance`

Example:

.. code-block:: bash

   python experiments/01_end_to_end/codes/eval_distributional.py \
     data=mydata \
     metric=_distributional \
     metric.evaluator.emb_zarr_path=null \
     'model=eval/<model>' \
     'model.load.model_name="<run_name>"' \
     'model.load.path="/path/to/model.ckpt"' \
     train_fraction=0 \
     test_fraction=1 \
     seed=42 \
     'split=<split>' \
     'output.save_path=/path/to/results/mydata/<model>_distributional.csv'

**Biological metrics.**

Available metrics: :func:`~sc_reconstruction.metrics.metric_cellcycle`,
:func:`~sc_reconstruction.metrics.metric_pathway`,
:func:`~sc_reconstruction.metrics.metric_coexpression`,
:func:`~sc_reconstruction.metrics.metric_deg`, and
:func:`~sc_reconstruction.metrics.metric_cytokine`.

The coexpression example below uses MSigDB Hallmark gene sets retrieved
through Omnipath. The expression store's root ``var_names`` attribute must
contain gene symbols matching those annotations.

Example:

.. code-block:: bash

   python experiments/01_end_to_end/codes/eval_biological.py \
     data=mydata \
     metric=_coexpression \
     metric.evaluator.mode=reconstruction \
     metric.evaluator.emb_zarr_path=null \
     'model=eval/<model>' \
     'model.load.model_name="<run_name>"' \
     'model.load.path="/path/to/model.ckpt"' \
     train_fraction=0 \
     test_fraction=1 \
     seed=42 \
     'split=<split>' \
     'output.save_path=/path/to/results/mydata/<model>_coexpression.csv'

For another biological metric, select its configuration under
``experiments/01_end_to_end/configs/metric/`` and supply its required
references or annotations; see the :doc:`Metrics tutorial <../tutorials/metrics>`.

4.3. Outputs
^^^^^^^^^^^^

Each command saves results to ``output.save_path``.
Rerunning a command replaces the existing output.

5. Extract embeddings (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Extract embeddings from a trained model for downstream tasks (e.g. training perturbation-response models such as STATE or CellFlow; 
see :ref:`Latent shift <custom-data-latent-shift>`) using the inputs prepared in :ref:`section 1.4 <custom-data-e2e-embedding-inputs>`.

5.1. Configure
^^^^^^^^^^^^^^

.. dropdown:: Embedding extraction parameters

   .. tab-set::

      .. tab-item:: Common settings

         .. csv-table::
            :header: "Flag", "Required / optional", "Meaning"
            :widths: 30, 30, 40

            "``--model``", "Required", "``AE`` (including olAE and mlAE), ``nlscVI``, or ``PCA``"
            "``--hvg_zarr``", "Required for custom data", "Path to the grouped ``expression.zarr`` store"
            "``--out``", "Required", "Output Zarr path; use a separate path for each trained model and split"

      .. tab-item:: AE and nlscVI

         .. csv-table::
            :header: "Flag", "Required / optional", "Meaning"
            :widths: 30, 30, 40

            "``--ckpt``", "Required for AE/nlscVI", "Trained checkpoint: ``.ckpt`` for AE variants or ``.pt`` for nlscVI"
            "``--device``", "Optional; default: ``cuda``", "``cpu`` or ``cuda`` for AE/nlscVI"
            "``--batch_size``", "Optional; default: 4096", "Cells per encoding batch for AE/nlscVI"

      .. tab-item:: PCA

         .. csv-table::
            :header: "Flag", "Required / optional", "Meaning"
            :widths: 30, 30, 40

            "``--pca_mean``", "Required for PCA", "Path to the saved ``mean.zarr``"
            "``--pca_pc``", "Required for PCA", "Path to the saved components; retain the filename ``pc_<n_components>.zarr``"

5.2. Run
^^^^^^^^

Run from the repository root in the ``reconeval`` environment, or follow
:ref:`Run through SLURM <slurm-parameters>`.

Example for AE:

.. code-block:: bash

   python experiments/01_end_to_end/codes/extract_e2e_embeddings.py \
     --model AE \
     --ckpt /path/to/model.ckpt \
     --hvg_zarr /path/to/mydata/expression.zarr \
     --out /path/to/mydata/embeddings/AE_emb.zarr \
     --device cpu

5.3. Outputs
^^^^^^^^^^^^

Embeddings are saved as cells-by-latent-dimensions arrays under
``<group>/X`` in the Zarr store specified by ``--out``.

For preparation of latent-shift inputs, follow
:ref:`Latent-shift inputs <custom-data-latent-inputs>`.
