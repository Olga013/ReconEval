Use custom data
===============

Choose a task:

- **End-to-end reconstruction**: train an encoder and decoder on your data
- **Foundation model**: train a decoder on pretrained foundation-model
  embeddings
- **Latent shift**: train a perturbation predictor on precomputed
  embeddings, then map its predictions to gene expression using the
  corresponding pretrained decoder

Follow these steps for your chosen task:

- :ref:`Set up the environment <custom-data-installation>`
- Prepare your **preprocessed** input data
- Choose and train a model via Python CLI or :ref:`SLURM <slurm-parameters>`
- Prepare a :ref:`reference projector <custom-data-e2e-reference-pca>` (if applicable)
- Evaluate reconstruction on held-out data
- Optionally extract embeddings for downstream tasks

The examples use ``mydata`` as the dataset name and ``/path/to/...`` for
paths; replace these placeholders with your own values.

.. toctree::
   :maxdepth: 1

   custom_datasets/prerequisites
   custom_datasets/end_to_end
   custom_datasets/foundation_model
   custom_datasets/latent_shift
   custom_datasets/slurm
