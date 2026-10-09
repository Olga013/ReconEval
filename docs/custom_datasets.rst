Use custom data
===============

Choose a task:

- **End-to-end reconstruction**: train an encoder and decoder on your data
- **Foundation model**: train a decoder on pretrained foundation-model
  embeddings
- **Latent shift**: train a perturbation predictor on precomputed
  embeddings, then map its predictions to gene expression using the
  corresponding pretrained decoder

**Outline**

- :ref:`Set up the environment <custom-data-installation>`
- Prepare your **preprocessed** input data
- Choose and train a model via Python CLI or :ref:`SLURM <slurm-parameters>`
- Prepare a :ref:`reference projector <custom-data-e2e-reference-pca>` (if applicable)
- Evaluate reconstruction on held-out data
- Optionally extract embeddings for downstream tasks

The examples use ``mydata`` as the dataset name and ``/path/to/...`` for
paths; replace these placeholders with your own values.

.. _custom-data-installation:

1. Set up prerequisites
-----------------------

To create environment check the following instructions:

- Environments `README <https://github.com/theislab/ReconEval/blob/main/envs/README.md>`_ in the ReconEval repository
- :doc:`installation`

.. _custom-data-end-to-end:

2. End-to-end reconstruction
----------------------------

For a worked example, see the
:doc:`End-to-end tutorial <tutorials/end_to_end>`.

2.1. Prepare input data
~~~~~~~~~~~~~~~~~~~~~~~

Provide data for the chosen workflow:

- For training:
  
  - expression per split (train, val, test)

- For evaluation and embedding extraction:

  - a grouped expression store
  - split metadata (evaluation only)

The preprocessing example on the LuCA dataset: ``experiments/preprocessing/preprocess_luca.py``.

.. _custom-data-e2e-training-inputs:

2.1.1. Training
^^^^^^^^^^^^^^^

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

2.1.2. Reference PCA
^^^^^^^^^^^^^^^^^^^^

Combine measured training, validation, and test expression in
``/path/to/mydata/reference.h5ad``, including each cell once.
Store a cells-by-genes CSR matrix in ``.X``, preserving evaluation
preprocessing and gene order.

Use this file only to fit the reference PCA.

.. _custom-data-e2e-eval-inputs:

2.1.3. Evaluation
^^^^^^^^^^^^^^^^^

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


.. note::

  For held-out evaluation set ``train_fraction=0`` and ``test_fraction=1`` in configuration 
  to evaluate only groups listed in ``test_combinations``.

.. note::

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

2.1.4. Embedding extraction
^^^^^^^^^^^^^^^^^^^^^^^^^^^
Use your trained model to obtain embeddings from ``expression.zarr`` prepared in :ref:`section 2.1.3 <custom-data-e2e-eval-inputs>`.
The extractor processes all groups in the store; split metadata is not required.

.. _custom-data-e2e-data-config:

2.1.5. Data configuration
^^^^^^^^^^^^^^^^^^^^^^^^^

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

2.2. Train
~~~~~~~~~~

2.2.1. Configure training
^^^^^^^^^^^^^^^^^^^^^^^^^

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
  `Bringing your own model <tutorials/end_to_end.html#bringing-your-own-model>`_)

Reuse the dataset configuration prepared in
:ref:`section 2.1.5 <custom-data-e2e-data-config>`.

Set the parameters in YAML or pass them as ``key=value`` on the CLI:

.. csv-table::
   :header: "Parameter", "Required / optional", "Value / meaning"
   :widths: 40, 25, 35

   "**Common settings**", "", ""
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
   "**AE variants**", "", ""
   "``model.meta.name``", "Required for olAE/mlAE; default: ``AE``", "``AE``, ``olAE``, or ``mlAE``; selects the dataloader settings and labels model outputs"
   "``model.model_args.library_size_mode``", "Required for olAE/mlAE; default: ``none``", "``none``: **AE** - no library-size scaling, default; ``observed``: **olAE** - AE with observed library size; ``modeled``: **mlAE** - AE with learned library size"
   "``model.model_args.n_hidden``", "Optional; default: ``[1024]``", "Hidden-layer widths; list length determines depth"
   "``model.model_args.n_latent``", "Optional; default: ``100``", "Latent dimension"
   "**VAE variants**", "", ""
   "``model.model_args.n_hidden``", "Optional; default: ``1024``", "Hidden-layer width"
   "``model.model_args.n_latent``", "Optional; default: ``300``", "Latent dimension"
   "``model.model_args.n_layers``", "Optional; default: ``3``", "Number of hidden layers"
   "``model.model_args.use_observed_lib_size``", "Optional; scVI/nlscVI default: ``true``; mlscVI default: ``false``", "``true`` uses observed totals; ``false`` infers library size"
   "**PCA**", "", ""
   "``model.model_args.n_components``", "Optional; default: ``300``", "Number of principal components to fit"

Defaults above refer to the provided data template and training configurations.

.. note::

  For AE variants, the ``model.meta.name`` label must match the ``model.model_args.library_size_mode`` behavior.

.. _custom-data-e2e-pca-setup:

.. note::

   Before running PCA, replace the default ``temp_dir`` in
   ``ReconPCA._setup_cluster``
   (``src/sc_reconstruction/models/reconpca.py``) with a writable
   directory for temporary files on your machine.

.. _custom-data-e2e-cli:

2.2.2. Run from Python CLI
^^^^^^^^^^^^^^^^^^^^^^^^^^

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

.. note::

   - Both options use Hydra's ``key=value`` syntax
   - ``data=mydata`` selects ``configs/data/mydata.yaml``
   - Command-line values take precedence over YAML

2.2.3. Outputs
^^^^^^^^^^^^^^

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

The saved files depend on the selected model:

- **AE variants:** ``.ckpt`` checkpoints
- **scVI variants:** ``.pt`` checkpoints
- **PCA:** ``mean.zarr`` and ``pc_<n_components>.zarr``, containing
  the training mean and principal-component matrix. Both are needed
  for reconstruction

.. note::

   AE and scVI variants save checkpoints beneath ``<model.save.dir>/<model.save.filename>/``. 
   The ``model.save.filename`` setting names the run directory.

   For AE, the default format is
   ``<max_epochs>_<n_hidden>_<n_latent>_<YYYYMMDD>``. For example:

   .. code-block:: text

      .../weights/mydata/split03/AE/Default/100_[1024]_128_20261007/

   For scVI variants, the directory name also includes ``n_layers`` and
   ``max_kl_weight``.

**Logs and configuration**

Using the same output root:

- ``logs/`` is the configured W&B logging directory
- ``outputs/<dataset>/<model>/<YYYY-MM-DD_HH-MM>/`` contains Hydra's
  run files, including configuration and overrides under ``.hydra/``

.. _custom-data-e2e-reference-pca:

2.3. Prepare the reference PCA
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

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

2.4. Evaluate
~~~~~~~~~~~~~
The evaluation consists of two steps:

- Reconstruct the held-out data
- Score the measured and reconstructed expression

To explore a toy example, see the :doc:`End-to-end tutorial <tutorials/end_to_end>`.
This section explains how to configure and run the evaluation scripts from the Python CLI.

For end-to-end evaluation, the framework supports:

- Statistical metrics
- Biological metrics

For details, see `Metric families <overview.html#metric-families>`_ and the
:doc:`Metrics tutorial <tutorials/metrics>`.

2.4.1. Configure evaluation
^^^^^^^^^^^^^^^^^^^^^^^^^^^

For the evaluation scripts, reuse the dataset configuration from
:ref:`section 2.1.5 <custom-data-e2e-data-config>`, select a
metric configuration under ``experiments/01_end_to_end/configs/metric/``,
and create a model evaluation YAML under ``experiments/01_end_to_end/configs/model/eval/``.

Example for AE (``model/eval/AE.yaml``):

.. code-block:: yaml

   meta:
     name: AE

   load:
     path: "/path/to/weights/mydata/<split>/AE/Default/<max_epochs>_<n_hidden>_<n_latent>_<YYYYMMDD>/epoch=<epoch>-val/loss_epoch=<val_loss>.ckpt"
     model_name: "<max_epochs>_<n_hidden>_<n_latent>_<YYYYMMDD>"  # e.g. 100_[1024]_128_20261007
     additional_params:
       distribution: normal
       input_dim: ${data.input_dim}

Use the exact checkpoint path and the matching run-directory name; see :ref:`Model files <custom-data-e2e-model-files>`.

Set the parameters in YAML or pass them as ``key=value`` on the CLI:

.. csv-table::
   :header: "Parameter", "Required / optional", "Value / meaning"
   :widths: 40, 25, 35

   "**Common settings**", "", ""
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
   "**AE variants**", "", ""
   "``model.load.additional_params.input_dim``", "Required for dynamic AE construction", "``${data.input_dim}``; match the training gene count"
   "``model.load.additional_params.distribution``", "Optional; default: ``normal``", "Match the distribution used during training"
   "``model.load.additional_params.library_size_mode``", "Optional; default: ``none``", "Match the trained variant: AE ``none``, olAE ``observed``, mlAE ``modeled``"
   "**VAE variants**", "", ""
   "``model.load.additional_params.gene_likelihood``", "Optional; match training", "Likelihood used to train scVI, nlscVI, or mlscVI"
   "``model.load.additional_params.use_observed_lib_size``", "Optional; match training", "Current training defaults: scVI/nlscVI ``true``; mlscVI ``false``"
   "**PCA**", "", ""
   "``model.model_args._target_``", "Required in the PCA YAML", "``sc_reconstruction.models.reconpca.ReconPCA``"
   "**Metric settings**", "", ""
   "``metric.evaluator.mode``", "Required", "``reconstruction``"
   "``metric.evaluator.emb_zarr_path``", "Optional", "Omit or set to ``null`` for reconstruction"
   "**Reference PCA**", "", ""
   "``metric.projector.load.path``", "Required for distributional evaluation", "Ordered paths to the saved reference mean and components; reuse the reference PCA across models with matching preprocessing and gene order"

.. note::

   ``dynamic_model_loading`` is ``true`` by default in ``base_eval.yaml``.

   - ``true``: initialize from ``model_args`` if provided. Otherwise, infer
     initial model parameters from ``load.model_name`` and combine them with
     ``load.additional_params``. Additional parameters override inferred values
   - ``false``: initialize from ``model_args`` directly;
     ``load.additional_params`` is ignored

.. note::

   For distributional evaluation, use the files from
   :ref:`Prepare the reference PCA <custom-data-e2e-reference-pca>`.
   Create ``experiments/01_end_to_end/configs/metric/projector/PCA.yaml``
   with these paths:

   .. code-block:: yaml

      model_args:
        _target_: sc_reconstruction.models.reconpca.ReconPCA

      load:
        path:
          - /path/to/reference_pca/mean.zarr
          - /path/to/reference_pca/pc_50.zarr

2.4.2. Run from Python CLI
^^^^^^^^^^^^^^^^^^^^^^^^^^

Run from the repository root in the ``reconeval`` environment, or follow
:ref:`Run through SLURM <slurm-parameters>`.

Replace ``<model>`` with the name of the model evaluation YAML created in
section 2.4.1. The templates assume that ``data/mydata.yaml`` and
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
references or annotations; see the :doc:`Metrics tutorial <tutorials/metrics>`.

2.4.3. Outputs
^^^^^^^^^^^^^^

Each command saves results to ``output.save_path``.
Rerunning a command replaces the existing output.

2.5. Extract embeddings (optional)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Extract embeddings from a trained model for downstream tasks (e.g. training perturbation-response models such as STATE or CellFlow; 
see :ref:`section 4 <custom-data-latent-shift>`) using the inputs prepared in :ref:`section 2.1.4 <custom-data-e2e-embedding-inputs>`.

2.5.1. Configure extraction
^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. csv-table::
   :header: "Flag", "Required / optional", "Meaning"
   :widths: 30, 30, 40

   "``--model``", "Required", "``AE`` (including olAE and mlAE), ``nlscVI``, or ``PCA``"
   "``--ckpt``", "Required for AE/nlscVI", "Trained checkpoint: ``.ckpt`` for AE variants or ``.pt`` for nlscVI"
   "``--hvg_zarr``", "Required for custom data", "Path to the grouped ``expression.zarr`` store"
   "``--out``", "Required", "Output Zarr path; use a separate path for each trained model and split"
   "``--pca_mean``", "Required for PCA", "Path to the saved ``mean.zarr``"
   "``--pca_pc``", "Required for PCA", "Path to the saved components; retain the filename ``pc_<n_components>.zarr``"
   "``--device``", "Optional; default: ``cuda``", "``cpu`` or ``cuda`` for AE/nlscVI"
   "``--batch_size``", "Optional; default: 4096", "Cells per encoding batch for AE/nlscVI"

2.5.2. Run from Python CLI
^^^^^^^^^^^^^^^^^^^^^^^^^^

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

2.5.3. Outputs
^^^^^^^^^^^^^^

Embeddings are saved as cells-by-latent-dimensions arrays under
``<group>/X`` in the Zarr store specified by ``--out``.

For preparation of latent-shift inputs, follow
:ref:`section 4.1 <custom-data-latent-inputs>`.

1. Foundation model
------------------------------------------------

This section uses an MLP decoder, which achieved the highest overall
score in our foundation-model reconstruction benchmark on PBMC-10M.

.. _custom-data-mlp-data:

3.1. Add embeddings and expression targets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Generate embeddings following instructions in :doc:`tutorials/fm`.
   Each model requires its own environment (see
   `README <https://github.com/theislab/ReconEval/blob/main/envs/README.md>`_
   in the ReconEval repository)

2. Provide the following files for MLP decoder:

.. code-block:: text

   /path/to/mydata/
   ├── embeddings/
   │   ├── train.zarr/
   │   └── val.zarr/
   ├── all_genes.zarr/
   └── target_genes.zarr/

.. note::

   Each of ``.zarr`` files should contain two arrays: expression and
   embeddings under ``SE``, ``scGPT``, ``scConcept``, or ``SCimilarity``.

.. _custom-data-mlp-cli:

3.2. Run from CLI
~~~~~~~~~~~~~~~~~

For cluster submission, follow :ref:`Run through SLURM <slurm-parameters>`.

**Run the MLP decoder:**

1. Activate ``reconeval`` defined in :ref:`installation step <custom-data-installation>`

2. Set the paths and embedding key, then train the decoder. Decoder training does not require YAML configuration file. For ``SE`` embeddings:

   .. code-block:: bash

      python experiments/02_foundation_model/codes/train_decoder_from_embedding.py \
        --emb-train-zarr /path/to/mydata/embeddings/train.zarr \
        --emb-val-zarr /path/to/mydata/embeddings/val.zarr \
        --all-genes-zarr /path/to/mydata/all_genes.zarr \
        --target-genes-zarr /path/to/mydata/target_genes.zarr \
        --embedding-key SE \
        --num-workers 0 \
        --epochs 500 \
        --out /path/to/results/mydata/SE/MLP

**Required inputs and common optional settings:**

.. csv-table::
   :header: "Flag", "Required / optional", "Meaning"
   :widths: 30, 30, 40

   "``--emb-train-zarr``", "Required", "Training store containing expression and embeddings"
   "``--emb-val-zarr``", "Required", "Validation store containing expression and embeddings"
   "``--all-genes-zarr``", "Required", "Store whose var_names attribute lists the expression columns in order"
   "``--target-genes-zarr``", "Required", "Store whose var_names attribute lists the genes to reconstruct"
   "``--embedding-key``", "Required", "Embedding array name: SE, scGPT, scConcept, or scimilarity"
   "``--out``", "Required", "Output directory for checkpoints and configuration"
   "``--epochs``", "Optional; default: 500", "Maximum training epochs"
   "``--batch-size``", "Optional; default: 256", "Cells per batch"
   "``--lr``", "Optional; default: 0.0001", "Learning rate"
   "``--n-layers``", "Optional; default: 1", "Number of hidden layers"
   "``--hidden``", "Optional; default: 4096", "Hidden-layer width"
   "``--num-workers``", "Optional; default: 16", "Data-loading workers; use 0 to load data in the main process"
   "``--seed``", "Optional; default: 42", "Random seed"

3.3. Outputs
~~~~~~~~~~~~

**Model files.**

Training saves model outputs in the directory specified by ``--out``,
such as ``/path/to/results/mydata/SE/MLP/``.

- **MLP:** a ``.ckpt`` checkpoint with the lowest validation loss

**Logs and configuration.**

- ``config.yaml`` in the same directory records the run settings, including
  the embedding dimension (``latent_dim``) and number of output genes
  (``gene_dim``)

.. _custom-data-latent-shift:

4. Latent shift
--------------------------------------------------

For STATE and CellFlow, generate embeddings using a model trained in
:ref:`End-to-end <custom-data-end-to-end>` or a pretrained foundation model
(:ref:`embedding preparation <custom-data-mlp-data>`). Use the same encoder for all
splits and save embeddings in ``.obsm`` (e.g. ``.obsm["X_AE_128"]``).

.. _custom-data-latent-inputs:

4.1. Add expression, embeddings, and conditions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Choose a representation and width from the training script's ``EMB_DIMS``
   and prepare the corresponding embeddings (e.g. ``AE_128``)
2. Provide the following files for STATE and CellFlow:

.. code-block:: text

   /path/to/mydata/
   ├── train/train.h5ad
   ├── val/val.h5ad
   └── test/test.h5ad

.. note::

   Each ``.h5ad`` file should contain:

   - Expression in ``.X`` and embeddings in ``.obsm["X_AE_128"]`` (for ``AE_128``),
     with matching cell order. Expression genes must follow the order expected
     by the decoder
   - Metadata in ``.obs["donor"]``, ``.obs["cell_type"]``, and
     ``.obs["target_gene"]``
   - Controls labelled ``PBS`` in ``.obs["target_gene"]`` for the
     donor/cell-type groups being predicted

   Adapt the metadata keys and control labels to your dataset.

.. _custom-data-state-cli:

4.2. Run STATE from CLI
~~~~~~~~~~~~~~~~~~~~~~~

For cluster submission, follow :ref:`Run through SLURM <slurm-parameters>`.

**Run STATE:**

1. Activate ``reconeval-pancellflow`` from the
   :ref:`installation step <custom-data-installation>`

2. Copy ``template_train.toml`` and ``template_val.toml`` from
   ``experiments/03_latent_shift/configs/st/`` to ``mydata_train.toml`` and
   ``mydata_val.toml`` in the same directory. Set the split directories in
   ``[datasets]`` and use matching dataset names in ``[training]``.
   Keep the ``"train"`` value in both files; the script uses a separate
   data module's training loader for validation

3. In ``experiments/03_latent_shift/codes/train_st.py``, point
   ``build_data_module`` and ``build_val_data_module`` to those TOML files.
   Update ``DATA_KWARGS_COMMON`` for your metadata keys and control label

4. Set the representation, decoder checkpoint, and output directory, then
   train STATE. The checkpoint must match the embedding width and output
   gene order. For ``AE_128`` embeddings:

   .. code-block:: bash

      python experiments/03_latent_shift/codes/train_st.py \
        --model AE_128 \
        --decoder_mode frozen \
        --decoder_ckpt /path/to/ae128.ckpt \
        --max_steps 40000 \
        --batch_size 16 \
        --no_wandb \
        --out_root /path/to/results/mydata/state

**Required inputs and common optional settings:**

.. csv-table::
   :header: "Flag", "Required / optional", "Meaning"
   :widths: 30, 30, 40

   "``--model``", "Required", "Representation listed in EMB_DIMS, e.g. AE_128"
   "``--decoder_mode``", "Optional; default: frozen", "Use a trained decoder; fresh initializes one"
   "``--decoder_ckpt``", "Optional; default: path in DECODER_CONFIGS", "Checkpoint for a frozen AE, VAE, or MLP decoder; ignored for PCA and fresh decoders"
   "``--decoder_weight``", "Optional; default: 0.0", "Gene-space loss weight; use a positive value to train a fresh decoder"
   "``--max_steps``", "Optional; default: 40000", "Maximum training steps"
   "``--batch_size``", "Optional; default: 16", "Cell sets per training batch"
   "``--lr``", "Optional; default: 0.0001", "Learning rate"
   "``--seed``", "Optional; default: 42", "Random seed"
   "``--arch_config``", "Optional; default: hf_se_parse", "Architecture YAML name under configs/arch, or a YAML file path"
   "``--cell_set_len``", "Optional; default: architecture setting (512 for hf_se_parse)", "Cells per set"
   "``--no_wandb``", "Optional; default: false", "Pass this flag to disable W&B logging"
   "``--out_root``", "Optional; default: script's OUT_ROOT", "Output root; a run subdirectory is created. Set a writable path for your run"

For PCA, set the decoder's ``mean_path`` and ``pc_path`` in
``DECODER_CONFIGS``.

.. _custom-data-cellflow-cli:

4.3. Run CellFlow from CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~

For cluster submission, follow :ref:`Run through SLURM <slurm-parameters>`.

**Run CellFlow:**

1. Activate ``reconeval-pancellflow`` from the
   :ref:`installation step <custom-data-installation>`

2. In ``experiments/03_latent_shift/codes/train_cf.py``, set ``DATA_ROOT``
   to ``/path/to/mydata`` and adapt the metadata/control labels if needed

3. In ``train_cf.py``, update the block that calls ``hf_hub_download()``
   with ``filename="pbmc_parse.h5ad"`` and reads ``uns/esm2_embeddings``
   to load your dataset's perturbation features. Retain a vector for every
   non-control perturbation across all splits, including any absent from
   training. Validation/test donor and cell-type labels must occur in the
   training categories

4. Set the representation and output directory, then train CellFlow.
   For ``AE_128`` embeddings:

   .. code-block:: bash

      python experiments/03_latent_shift/codes/train_cf.py \
        --model AE_128 \
        --config repro \
        --num_iters 500000 \
        --batch_size 1024 \
        --valid_freq 50000 \
        --out_dir /path/to/results/mydata/cellflow

**Required inputs and common optional settings:**

.. csv-table::
   :header: "Flag", "Required / optional", "Meaning"
   :widths: 30, 30, 40

   "``--model``", "Required", "Representation listed in EMB_DIMS, e.g. AE_128"
   "``--config``", "Optional; default: repro", "Training preset: repro or paper"
   "``--num_iters``", "Optional; default: 500000", "Training iterations"
   "``--batch_size``", "Optional; default: 1024", "Cells per batch"
   "``--valid_freq``", "Optional; default: 50000", "Validation interval in iterations"
   "``--seed``", "Optional; default: 42", "Random seed"
   "``--out_dir``", "Optional; default: generated under out_root", "Exact run output directory; set a writable path for your run"
   "``--out_root``", "Optional; default: script's OUT_ROOT", "Output root used when out_dir is omitted"

.. note::

   During training, metrics labelled ``test`` are computed on ``val/val.h5ad``.

4.4. Outputs
~~~~~~~~~~~~

**Model files**

- **STATE:** saves models under ``<out_root>/<run_name>/checkpoints/``,
  including the three best validation checkpoints, ``last.ckpt``, and
  ``final.ckpt``. The run name retains ``pbmc_split03`` even for custom
  data. An existing ``last.ckpt`` is used to resume training automatically
- **CellFlow:** saves the best and last CellFlow model artifacts in the
  directory specified by ``--out_dir``. The best model is selected using
  validation data

Set ``--out_root`` for STATE and ``--out_dir`` for CellFlow to writable
paths, as in the CLI examples. Their defaults use cluster-specific paths;
``RECONEVAL_OUT`` does not redirect these outputs.

**Logs and configuration**

- **STATE:** ``config.yaml`` and label mappings are saved in the run
  directory, alongside ``checkpoints/``
- **CellFlow:** ``config.json``, ``training_logs.json``, and
  ``training_curves.png`` are saved in the output directory

.. _slurm-parameters:

5. Run through SLURM (optional)
-------------------------------

Use these steps to submit any task's Python command as a cluster job.
Complete the data and configuration steps for your task first.
Skip this section for direct Python runs.

1. Create a ``.sbatch`` file starting with ``#!/bin/bash``, using a script
   from ``experiments/*/submit/`` as a reference. Set the ``#SBATCH``
   options for your cluster and workload: partition/account, time limit,
   CPUs, memory, and GPUs if needed. Existing resource requests were
   chosen for the benchmark datasets

2. Set the output and error log paths in the script, for example:

   .. code-block:: bash

      #SBATCH --output=logs/slurm/%x-%j.out
      #SBATCH --error=logs/slurm/%x-%j.err

   Place all ``#SBATCH`` directives before shell commands. Here ``%x``
   is the job name and ``%j`` is the job ID

3. Initialize Conda, activate the environment specified in your task's
   CLI instructions, and change to the repository directory:

   .. code-block:: bash

      source /path/to/miniforge3/etc/profile.d/conda.sh
      conda activate your-task-environment
      cd /path/to/ReconEval

   Replace the paths and environment name with your local setup

4. Add the complete Python command from your task's CLI section:
   :ref:`End-to-end <custom-data-e2e-cli>`,
   :ref:`Foundation model <custom-data-mlp-cli>`,
   :ref:`STATE <custom-data-state-cli>`, or
   :ref:`CellFlow <custom-data-cellflow-cli>`.
   Set your input paths, output directory, and model options there.
   When adapting an existing submission script, replace its training
   command with your own

5. Create the log directory **before** submission and submit from the
   repository root:

   .. code-block:: bash

      cd /path/to/ReconEval
      mkdir -p logs/slurm
      sbatch /path/to/train_mydata.sbatch

   Adjust the log directory to match your ``#SBATCH`` paths. The log
   files contain the job's standard output and errors; model outputs
   are saved at the locations described in each task's **Outputs** section

If you retain an existing script's environment-variable options, export
only variables that the script reads and submit with ``sbatch --export=ALL``.
Other settings belong in the Python command inside the script.
