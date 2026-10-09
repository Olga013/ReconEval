.. _custom-data-latent-shift:

Latent shift
------------

:doc:`Back to introduction <../custom_datasets>`

For STATE and CellFlow, generate embeddings using a model trained in
:ref:`End-to-end <custom-data-end-to-end>` or a pretrained foundation model
(:ref:`embedding preparation <custom-data-mlp-data>`). Use the same encoder for all
splits and save embeddings in ``.obsm`` (e.g. ``.obsm["X_AE_128"]``).

.. _custom-data-latent-inputs:

1. Add expression, embeddings, and conditions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Choose a representation and width from the training script's ``EMB_DIMS``
   and prepare the corresponding embeddings (e.g. ``AE_128``)
2. Provide the following files for STATE and CellFlow:

.. code-block:: text

   /path/to/mydata/
   ├── train/train.h5ad
   ├── val/val.h5ad
   └── test/test.h5ad

.. dropdown:: Note: Details on latent shift inputs
   :icon: info
   :color: info
   :animate: fade-in-slide-down

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

2. Run STATE from CLI
~~~~~~~~~~~~~~~~~~~~~

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

.. dropdown:: Training parameters

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

3. Run CellFlow from CLI
~~~~~~~~~~~~~~~~~~~~~~~~

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

.. dropdown:: Training parameters

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

.. dropdown:: Note: CellFlow validation metrics
   :icon: info
   :color: info
   :animate: fade-in-slide-down

   During training, metrics labelled ``test`` are computed on ``val/val.h5ad``.

4. Outputs
~~~~~~~~~~

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
