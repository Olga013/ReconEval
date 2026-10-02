Use custom data
===============

- Start with a **preprocessed** dataset
- Choose the task below:
  
  - 01_end_to_end
  - 02_foundation_model
  - 03_latent_shift
  
- Run:
   
  - Python CLI directly
  - or submit it through SLURM
  
- The current documentation uses ``mydata`` and ``/path/to/...`` as the examples for dataset name 
  and path to it respectively; replace these placeholders.

.. _custom-data-installation:

1. Set up prerequisites
-----------------------

To create environment check the following instructions:

- Environments `README <https://github.com/theislab/ReconEval/blob/main/envs/README.md>`_ in the ReconEval repository
- :doc:`installation`

**NB!** For end-to-end training, choose an output root and optionally use offline W&B logging:

.. code-block:: bash

   export RECONEVAL_OUT=/path/to/results
   export WANDB_MODE=offline

The standalone MLP, STATE, and CellFlow scripts use their own output flags,
shown below.

.. _slurm-parameters:

2. Set up SLURM parameters
--------------------------

Skip this section for direct Python runs.

Before submitting a job:

1. Use scripts located in ``experiment/*/submit/`` directory
   as the examples for your own SLURM submission scripts.

2. Edit ``#SBATCH`` settings in your ``.sbatch`` file for your cluster. Existing resource requests
   were used for the benchmark datasets, use them as a reference.

3. Update the Conda initialization and activation lines to use your local
   Conda setup and the environment created in the
   :ref:`installation step <custom-data-installation>`:

   .. code-block:: bash

      source /path/to/miniforge3/etc/profile.d/conda.sh
      conda activate reconeval

4. Create the log directory **before** submission, from the repository root:

   .. code-block:: bash

      mkdir -p logs/slurm


**NB!** Each task includes the variables supported by its submission scripts. 
You can configure each submission script using task-specific environment variables. 
To change other training settings, edit the Python command inside
your copied ``.sbatch`` file.

Check the AE training run with exported environment variables as an example:

.. code-block:: bash

   export DATA=mydata
   export LATENT=128
   export MAX_EPOCHS=100
   export MIN_EPOCHS=10

   sbatch --export=ALL experiments/01_end_to_end/submit/train_ae.sbatch


3. 01_end_to_end
----------------------------------------------

3.1. Add the preprocessed dataset
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- For AE and the scVI variants, provide:

  .. code-block:: text

     /path/to/mydata/
     ├── train.zarr
     ├── val.zarr
     └── test.zarr

- PCA instead reads ``train.h5ad`` from the same
  directory, with expression stored as a CSR matrix in ``.X``.


For AE and the scVI variants, each ``.zarr`` directory must contain a
dense expression matrix, with cells as rows and genes as columns.
Store this matrix at the root of the Zarr store or as an array named
``X``. For the default model configurations, use log-normalized
expression values stored as ``float32``.

Sparse AnnData Zarr exports cannot be read directly by this loader;
convert them to the dense layout described above before training.


3.2. Run from CLI
~~~~~~~~~~~~~~~~~~~~~~~~~

**Run with a YAML configuration:**

1. Use ``template.yaml`` and the example YAML files to create
   your own ``mydata.yaml`` configuration.
   Save it in ``experiments/01_end_to_end/configs/data/``.

2. Replace the required ``???`` fields and keep the
   model-specific dataloader settings:

   .. code-block:: yaml

      name: mydata
      path: /path/to/mydata
      input_dim: 2000

3. Run model training with your ``mydata.yaml`` configuration.

   For example, train an AE:

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

**NB!**

- Both options use Hydra's ``key=value`` syntax. 
- ``data=mydata`` selects ``configs/data/mydata.yaml``.
- Command-line values take precedence over YAML.

**Model selection:**

Available models:

- AE
- scVI
- nlscVI
- mlscVI
- PCA

**Common overrides:**

.. csv-table::
   :header: "Override", "Meaning"
   :widths: 50, 50

   "``data.name=mydata``", "Dataset label used in output paths and logs"
   "``data.path=/path/to/mydata``", "Directory containing the input split files"
   "``data.input_dim=2000``", "Number of expression columns in input data"
   "``data.model_specific.AE.minibatch_size=256``", "Cells per batch; replace ``AE`` with the selected non-PCA model."
   "``data.model_specific.AE.num_workers=0``", "Data-loading workers"
   "``trainer.max_epochs=100``", "Maximum epochs for AE/scVI variants"
   "``trainer.min_epochs=10``", "Minimum epochs"
   "``seed=42``", "Random seed"
   "``split=split03``", "A split label"

**NB!** PCA uses the GPU/RAPIDS implementation. Set a writable scratch directory
in ``ReconPCA._setup_cluster`` in
``src/sc_reconstruction/models/reconpca.py`` before running it on another
machine; its default is cluster-specific.

3.3. Run through SLURM
~~~~~~~~~~~~~~~~~~~~~~

1. Fill all required fields in ``mydata.yaml``. 
2. Set up SLURM parameters in your sbatch script as described in :ref:`section 2 <slurm-parameters>`.
3. Run the script with your dataset configuration, e.g. for AE:

.. code-block:: bash

   export DATA=mydata
   export LATENT=128
   export MAX_EPOCHS=100
   export MIN_EPOCHS=10

   sbatch --export=ALL experiments/01_end_to_end/submit/train_ae.sbatch


4. For another model, copy the corresponding script and use its supported
variables:

.. csv-table::
   :header: "Source script", "Environment variables and their defaults"
   :widths: 25, 75

   "``train_ae.sbatch``", "``DATA=tahoe``, ``LATENT=128``, ``MAX_EPOCHS=400``, ``MIN_EPOCHS=10``."
   "``train_scvi.sbatch``", "``DATA=tahoe``, ``MODEL=scVI``, ``SPLIT=split03``, ``LATENT=128``, ``N_HIDDEN=1024``, ``MAX_EPOCHS=200``, ``MIN_EPOCHS=1``."
   "``train_pca.sbatch``", "``DATA=tahoe``, ``SPLIT=split03``, ``LATENT=128`` (maps to ``n_components``)."

For ``train_scvi.sbatch``, ``MODEL`` also accepts ``nlscVI`` or ``mlscVI``.

3.4. Outputs
~~~~~~~~~~~~

Model artifacts are saved under
``$RECONEVAL_OUT/weights/mydata/<split>/<model>/<note>/``, or under
``~/reconeval_outputs/weights/`` when ``RECONEVAL_OUT`` is unset.
See :doc:`tutorials/end_to_end` for reconstruction using the Python API
and :doc:`tutorials/metrics` for scoring reconstructed expression.

1. 02_foundation_model
------------------------------------------------

4.1. Add embeddings and expression targets
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Generate embeddings with :doc:`tutorials/fm`, or use existing embeddings.
For the MLP decoder, provide:

.. code-block:: text

   /path/to/mydata/
   ├── embeddings/
   │   ├── train.zarr/
   │   └── val.zarr/
   ├── all_genes.zarr/
   └── target_genes.zarr/

- Each split contains dense ``float32`` arrays: expression in ``X`` and
  embeddings under ``SE``, ``scGPT``, ``scConcept``, or ``scimilarity``.
  Both arrays must use the same cell order.
- Use the same gene order across splits and log-normalized expression
  for the default loss.
- In ``attrs["var_names"]``, list the ``X`` genes in ``all_genes.zarr`` and
  the desired output genes in ``target_genes.zarr``, preserving their order.
  Target genes must be a subset of the full list. Split stores with these
  attributes can replace separate metadata stores.

**NB!** Export AnnData ``.obsm["X_fm"]`` embeddings to a top-level Zarr
array with one of the names above.

.. _custom-data-mlp-cli:

4.2. Run from CLI
~~~~~~~~~~~~~~~~~

**Run the MLP decoder:**

1. Activate ``reconeval`` from the
   :ref:`installation step <custom-data-installation>`.

2. Set the paths and embedding key, then train the decoder. For SE embeddings:

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

**Common settings:**

.. csv-table::
   :header: "Flag", "Meaning"
   :widths: 50, 50

   "``--epochs 500``", "Maximum training epochs"
   "``--batch-size 256``", "Cells per batch"
   "``--lr 0.0001``", "Learning rate"
   "``--n-layers 1`` / ``--hidden 4096``", "Hidden-layer count / width"
   "``--num-workers 0``", "Data-loading workers; default is 16"
   "``--seed 42``", "Random seed"

**NB!**

- This script uses ``--flag value`` arguments. All six data/output flags
  in the example are required; the embedding width is inferred.
- For small splits, use ``--num-workers 0`` and a ``--batch-size`` no larger
  than the smallest split; the loader emits full batches.
- The encoder stays frozen. Use ``--help`` for all decoder options.

4.3. Run through SLURM
~~~~~~~~~~~~~~~~~~~~~~

1. Prepare the Zarr files from section 4.1 and set the input paths,
   embedding key, and output directory in the command from
   :ref:`section 4.2 <custom-data-mlp-cli>`.

2. Use ``experiments/01_end_to_end/submit/train_ae.sbatch`` as a template
   for ``experiments/02_foundation_model/submit/train_mydata_mlp.sbatch``.
   Set up SLURM parameters as described in
   :ref:`section 2 <slurm-parameters>` and activate ``reconeval``.
   Replace the Python command with your command from section 4.2, preceded
   by ``cd /path/to/ReconEval``.

3. Submit your script from the repository root:

   .. code-block:: bash

      sbatch --export=ALL experiments/02_foundation_model/submit/train_mydata_mlp.sbatch

**NB!** Set training options in the Python command; this example does not
read them from environment variables. The existing ``decoderonly_grid.sbatch``
runs a separate benchmark-specific Hydra sweep.

4.4. Outputs
~~~~~~~~~~~~

``--out`` contains ``config.yaml`` and the checkpoint selected
by validation loss. To reconstruct held-out embeddings, instantiate
``ReconMLPDecoder(n_input=latent_dim, n_output=gene_dim)`` using the saved
config, call ``decoder.load("/path/to/model.ckpt")``, then
``decoder.decode(test_embeddings)`` with a NumPy array.
See :doc:`tutorials/fm` and :doc:`tutorials/metrics` for the Python workflow.
Other decoder training scripts are described in
``experiments/02_foundation_model/README.md`` and require their own configs.

5. 03_latent_shift
--------------------------------------------------

These scripts use existing representations. Custom data currently require
the data-path and metadata edits below; there is no dataset-path CLI flag
or ``data=mydata`` override for these scripts.

5.1. Add expression, embeddings, and conditions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For STATE and CellFlow, provide:

.. code-block:: text

   /path/to/mydata/
   ├── train/train.h5ad
   ├── val/val.h5ad
   └── test/test.h5ad

Each split needs (example: ``AE_128``):

- Expression in ``.X``, with the gene order expected by the decoder.
- Embeddings in ``.obsm["X_AE_128"]``, aligned with the expression rows.
- Metadata in ``.obs["donor"]``, ``.obs["cell_type"]``, and
  ``.obs["target_gene"]``.
- Controls labelled ``PBS`` for the donor/cell-type groups being predicted.

Use a representation and width listed in the script's ``EMB_DIMS``.
Adapt the metadata keys and control labels to your dataset.

5.2. Run STATE from CLI
~~~~~~~~~~~~~~~~~~~~~~~

Activate ``reconeval-pancellflow`` from the
:ref:`installation step <custom-data-installation>`.

**Configure and run:**

1. Copy ``pbmc_train.toml`` and ``pbmc_val.toml`` from
   ``experiments/03_latent_shift/configs/st/`` to ``mydata_train.toml`` and
   ``mydata_val.toml`` in the same directory. Set the ``[datasets]`` paths
   and matching dataset names in ``[training]``.

2. In ``experiments/03_latent_shift/codes/train_st.py``, point
   ``build_data_module`` and ``build_val_data_module`` to those TOML files.
   Update ``DATA_KWARGS_COMMON`` for your metadata and
   ``DECODER_CONFIGS["AE_128"]["ckpt"]`` for your AE checkpoint.
   The checkpoint must match the embedding width and output gene order.

3. Train STATE with AE-128 embeddings and a frozen decoder:

   .. code-block:: bash

      python experiments/03_latent_shift/codes/train_st.py \
        --model AE_128 \
        --decoder_mode frozen \
        --max_steps 40000 \
        --batch_size 16 \
        --no_wandb \
        --out_root /path/to/results/mydata/state

**Common settings:**

.. csv-table::
   :header: "Flag", "Meaning"
   :widths: 50, 50

   "``--max_steps 40000``", "Training-step limit"
   "``--batch_size 16``", "Cell sets per training batch"
   "``--lr 0.0001``", "Learning rate"
   "``--decoder_mode frozen``", "Use a trained decoder; ``fresh`` initializes one"
   "``--decoder_weight 0.0``", "Gene-space loss weight; use a positive value for a fresh decoder"
   "``--out_root PATH``", "Output root; a run subdirectory is created"

**NB!**

- Data paths are configured in the TOML files and script.
- ``--no_wandb`` disables W&B logging. Use ``--help`` for all options.

5.3. Run CellFlow from CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~

Use the same environment, ``reconeval-pancellflow``.

**Configure and run:**

1. In ``experiments/03_latent_shift/codes/train_cf.py``, set ``DATA_ROOT``
   to ``/path/to/mydata`` and adapt the metadata/control labels if needed.

2. Replace the PBMC-specific ESM2 lookup with your perturbation features.
   Provide a vector for every non-control perturbation. Validation/test
   donor and cell-type labels must occur in the training categories.

3. Train CellFlow with AE-128 embeddings:

   .. code-block:: bash

      python experiments/03_latent_shift/codes/train_cf.py \
        --model AE_128 \
        --config repro \
        --num_iters 500000 \
        --batch_size 1024 \
        --valid_freq 50000 \
        --out_dir /path/to/results/mydata/cellflow

**Common settings:**

.. csv-table::
   :header: "Flag", "Meaning"
   :widths: 50, 50

   "``--config repro``", "Training preset: ``repro`` or ``paper``"
   "``--num_iters 500000``", "Training iterations"
   "``--batch_size 1024``", "Cells per batch"
   "``--valid_freq 50000``", "Validation interval in iterations"
   "``--seed 42``", "Random seed"
   "``--out_dir PATH``", "Run output directory"

**NB!**

- Metrics labelled ``test`` use ``val/val.h5ad``; keep the test split held out.
- Use ``--help`` for all options.

5.4. Run through SLURM
~~~~~~~~~~~~~~~~~~~~~~

1. Complete the data and configuration edits in section 5.2 for STATE
   or section 5.3 for CellFlow.

2. Set up SLURM parameters in ``train_st.sbatch`` or ``train_cf.sbatch``
   under ``experiments/03_latent_shift/submit/``, as described in
   :ref:`section 2 <slurm-parameters>`. Set
   ``EXPDIR=/path/to/ReconEval/experiments/03_latent_shift`` and activate
   ``reconeval-pancellflow`` in your script.

3. Run STATE with your dataset configuration, e.g. for AE-128 embeddings:

   .. code-block:: bash

      export MODEL=AE_128
      export ARCH_CONFIG=hf_se_parse
      export DECODER_MODE=frozen
      export DECODER_WEIGHT=0.0
      export SEED=42
      export OUT_ROOT=/path/to/results/mydata/state

      sbatch --array=0 --export=ALL experiments/03_latent_shift/submit/train_st.sbatch

4. For CellFlow, use its submission script and supported variables:

   .. code-block:: bash

      export MODEL=AE_128
      export CONFIG=repro
      export NUM_ITERS=500000
      export BATCH_SIZE=1024
      export VALID_FREQ=50000
      export OUT_DIR=/path/to/results/mydata/cellflow

      sbatch --export=ALL experiments/03_latent_shift/submit/train_cf.sbatch

.. csv-table::
   :header: "Source script", "Environment variables and their defaults"
   :widths: 25, 75

   "``train_st.sbatch``", "``MODEL=PCA_32`` (array default), ``ARCH_CONFIG=paper_parse_pbmc``, ``DECODER_MODE=frozen``, ``DECODER_WEIGHT=0.0``, ``SEED=42``; optional ``OUT_ROOT`` and ``SCHEDULER_FLAG`` (both unset)."
   "``train_cf.sbatch``", "``MODEL=AE_2048``, ``CONFIG=repro``, ``NUM_ITERS=500000``, ``BATCH_SIZE=1024``, ``VALID_FREQ=50000``; optional ``OUT_DIR`` (unset)."

**NB!**

- The STATE example selects one model and one array index. Use
  ``export SCHEDULER_FLAG=--scheduler`` to enable learning-rate scheduling.
- STATE fixes ``--max_steps 200000``, ``--batch_size 16``, ``--lr 3e-4``,
  and ``--cell_set_len 512`` in its Python command. Edit that command to
  change them or add ``--no_wandb`` to disable W&B logging.
- To change CellFlow's seed, add ``--seed`` to its Python command;
  exporting ``SEED`` has no effect.

5.5. Find outputs and evaluate
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* **STATE:** ``--out_root/<run_name>/`` contains ``config.yaml``, label
  mappings, and ``checkpoints/`` with validation checkpoints,
  ``last.ckpt``, and ``final.ckpt``. The generated name currently retains
  the label ``pbmc_split03`` even for custom data. A run with an existing
  ``last.ckpt`` resumes automatically.
* **CellFlow:** ``--out_dir`` contains ``config.json``,
  ``training_logs.json``, ``training_curves.png``, and best/last CellFlow
  model artifacts.

``RECONEVAL_OUT`` does not redirect these scripts' outputs. For predictions
and public perturbational metrics, see :doc:`tutorials/latent_shift`;
replace its synthetic arrays with your own control, true-perturbed, and
predicted cells. The batch evaluators ``eval_cf.py`` and ``eval_st.py``
currently depend on ``sc_reconstruction.metrics.st_pert``, which is absent
from this checkout.
