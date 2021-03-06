# mT5: Multilingual T5

Multilingual T5 (mT5) is a massively multilingual pretrained text-to-text
transformer model, trained following a similar recipe as
[T5](https://github.com/google-research/text-to-text-transfer-transformer).
This repo can be used to reproduce the experiments in the [mT5 paper][paper].

## Table of Contents

* [Languages covered](#languages-covered)
* [Results](#results)
* [Usage](#usage)
  * [Training](#training)
  * [Fine-Tuning](#fine-tuning)
* [Released Model Checkpoints](#released-model-checkpoints)
* [How to Cite](#how-to-cite)

## Languages covered

mT5 is pretrained on the [mC4](https://www.tensorflow.org/datasets/catalog/c4#c4multilingual_nights_stay) corpus, covering 101 languages:

Afrikaans, Albanian, Amharic, Arabic, Armenian, Azerbaijani, Basque,
Belarusian, Bengali, Bulgarian, Burmese, Catalan, Cebuano, Chichewa, Chinese,
Corsican, Czech, Danish, Dutch, English, Esperanto, Estonian, Filipino,
Finnish, French, Galician, Georgian, German, Greek, Gujarati, Haitian Creole,
Hausa, Hawaiian, Hebrew, Hindi, Hmong, Hungarian, Icelandic, Igbo, Indonesian,
Irish, Italian, Japanese, Javanese, Kannada, Kazakh, Khmer, Korean, Kurdish,
Kyrgyz, Lao, Latin, Latvian, Lithuanian, Luxembourgish, Macedonian, Malagasy,
Malay, Malayalam, Maltese, Maori, Marathi, Mongolian, Nepali, Norwegian,
Pashto, Persian, Polish, Portuguese, Punjabi, Romanian, Russian, Samoan,
Scottish Gaelic, Serbian, Shona, Sindhi, Sinhala, Slovak, Slovenian, Somali,
Sotho, Spanish, Sundanese, Swahili, Swedish, Tajik, Tamil, Telugu, Thai,
Turkish, Ukrainian, Urdu, Uzbek, Vietnamese, Welsh, West Frisian, Xhosa,
Yiddish, Yoruba, Zulu.

## Results

mT5 achieves state-of-the-art performance on many
[XTREME](https://github.com/google-research/xtreme) tasks, as of October 2020.
For example, on zero-shot classification and QA tasks:

| Model | XNLI (acc.) | PAWS-X (acc.) | XQuAD (F1/EM) | MLQA (F1/EM) | TyDiQA-GoldP (F1/EM) |
| ---- | ---- | ---- | ---- | ---- | ---- |
| mBERT | 65.4 | 81.9 | 62.2 / 49.4 | 61.4 / 44.2 | 59.7 / 43.9 |
| XLM | 69.1 | 80.9 | 61.2 / 44.3 | 48.5 / 32.6 | 43.6 / 29.1 |
| InfoXLM | 81.4 | - | - / - | 73.6 / 55.2 | - / - |
| Phang et al. (2020) | 80.4 | 87.7 | 77.2 / 61.3 | 72.3 / 53.5 | 76.0 / 59.5 |
| XLM-R | 79.2 | 86.4 | 76.6 / 60.8 | 71.6 / 53.2 | 65.1 / 45.0 |
| mT5-Small | 67.5 | 82.4 | 58.1 / 42.5 | 54.6 / 37.1 | 34.9 / 23.9 |
| mT5-Base | 75.4 | 87.4 | 67.0 / 49.0 | 64.6 / 45.0 | 58.1 / 42.8 |
| mT5-Large | 81.1 | 89.6 | 77.8 / 61.5 | 71.2 / 51.7 | 57.8 / 41.1 |
| mT5-XL | 82.9 | **90.2** | 79.5 / 63.6 | 73.5 / 54.5 | 77.3 / 61.5 |
| mT5-XXL (75% trained) | **84.8** | 89.2 | **81.9 / 65.7** | **75.5 / 56.9** | **80.8 / 66.3** |

Check back here for updated results after our XXL model finishes training.

## Usage

### Training

To run this code, you need to install the
[t5 library](https://pypi.org/project/t5/). General instructions for training, fine-tuning, evaluation, and exporting models for inference can be found in the [t5 repo](https://github.com/google-research/text-to-text-transfer-transformer). In order to use the additional mT5 tasks provided in this library with the `t5_mesh_transformer commands`, run from this directory and add the flag `--module_import="multilingual_t5.tasks"`.

Example command to train a mT5-Large model on the [mc4](https://www.tensorflow.org/datasets/catalog/c4#c4multilingual_nights_stay) task from scratch as described in the paper.

```
export PROJECT=yourproject
export ZONE=yourzone
export BUCKET=yourbucket
export TPU=yourtpu

ctpu up   --name=$TPU   --project=$PROJECT  --zone=$ZONE   --tpu-size=v3-256   --tpu-only   --noconf

TASK=mc4
MODEL_DIR="${BUCKET}${TASK}"

t5_mesh_transformer \
  --tpu="${TPU}" \
  --gcp_project="${PROJECT}" \
  --tpu_zone="${ZONE}" \
  --model_dir="${MODEL_DIR}" \
  --gin_file="models/t5.1.1.large.gin" \
  --gin_file="objectives/span.gin" \
  --gin_param="MIXTURE_NAME = '${TASK}'" \
  --gin_param="inputs_length = 1024" \
  --gin_param="utils.run.batch_size = ('tokens_per_batch', 1048576)" \
  --gin_param="utils.run.learning_rate_schedule=@learning_rate_schedules.rsqrt_no_ramp_down" \
  --gin_param="run.train_steps = 1000000" \
  --gin_param="utils.tpu_mesh_shape.model_parallelism = 1" \
  --gin_param="utils.tpu_mesh_shape.tpu_topology = 'v3-256'" \
  --eval_gin_file="perplexity_eval.gin" \
  --gin_param="mesh_eval_dataset_fn.num_eval_examples = 10000" \
  --t5_tfds_data_dir="${BUCKET}/t5-tfds" \
  --module_import="multilingual_t5.tasks"
```

### Fine-Tuning

As an example to finetune `XNLI_zeroshot` task on mT5-Large model by running (from this directory):

```
export PROJECT=yourproject
export ZONE=yourzone
export BUCKET=yourbucket
export TPU=yourtpu

ctpu up   --name=$TPU   --project=$PROJECT  --zone=$ZONE   --tpu-size=v3-256   --tpu-only   --noconf

TASK=xnli_zeroshot
PRETRAINED_DIR=gs://t5-data/pretrained_models/mt5/large
PRETRAINED_STEPS=1000000
FINETUNE_STEPS=20000
MODEL_DIR="${BUCKET}${TASK}"

# Run fine-tuning
t5_mesh_transformer \
  --tpu="${TPU}" \
  --gcp_project="${PROJECT}" \
  --tpu_zone="${ZONE}" \
  --model_dir="${MODEL_DIR}" \
  --gin_file="${PRETRAINED_DIR}/operative_config.gin" \
  --gin_file="sequence_lengths/xnli_zeroshot.gin" \
  --gin_param="utils.tpu_mesh_shape.tpu_topology = 'v3-256'" \
  --gin_param="MIXTURE_NAME = '${TASK}'" \
  --gin_param="utils.run.train_steps=$((PRETRAINED_STEPS+FINETUNE_STEPS))" \
  --gin_param="utils.run.init_checkpoint='${PRETRAINED_DIR}/model.ckpt-${PRETRAINED_STEPS}'" \
  --t5_tfds_data_dir="${BUCKET}/t5-tfds" \
  --module_import="multilingual_t5.tasks" \
  --gin_location_prefix="multilingual_t5/gin/"
```

The remaining experiments are shown in the [tasks.py](multilingual_t5/tasks.py) file.

## Released Model Checkpoints

To facilitate reproducibility and future work, we have released the model checkpoints for pretraining models.

We have released the following checkpoints for pre-trained models described in our paper:

* **mT5-Small** (300 million parameters): [gs://t5-data/pretrained_models/mt5/small](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/mt5/small/)
* **mT5-Base** (600 million parameters): [gs://t5-data/pretrained_models/mt5/base](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/mt5/base/)
* **mT5-Large** (1 billion parameters): [gs://t5-data/pretrained_models/mt5/large](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/mt5/large/)
* **mT5-XL** (4 billion parameters): [gs://t5-data/pretrained_models/mt5/xl](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/mt5/xl/)
* **mT5-XXL** (13 billion parameters): [gs://t5-data/pretrained_models/mt5/xxl](https://console.cloud.google.com/storage/browser/t5-data/pretrained_models/mt5/xxl/)

# How to Cite

If you extend or use this work, please cite the [paper][paper] where it was
introduced:

```
@misc{xue2020mt5,
    title = {{mT5}: A massively multilingual pre-trained text-to-text transformer},
    author = {Linting Xue and Noah Constant and Adam Roberts and Mihir Kale and Rami Al-Rfou and Aditya Siddhant and Aditya Barua and Colin Raffel},
    year = {2020},
    eprint = {2010.11934},
    archivePrefix = {arXiv},
    primaryClass = {cs.CL}
}
```

[paper]: https://arxiv.org/abs/2010.11934
