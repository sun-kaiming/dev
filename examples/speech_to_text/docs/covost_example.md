[[Back]](..)

# S2T Example: ST on CoVoST

We replicate the experiments in
[CoVoST 2 and Massively Multilingual Speech-to-Text Translation (Wang et al., 2020)](https://arxiv.org/abs/2007.10310).

## Data Preparation

[Download](https://commonvoice.mozilla.org/en/datasets) and unpack Common Voice v4 to a path
`${COVOST_ROOT}/${SOURCE_LANG_ID}`, then preprocess it with

```bash
# additional Python packages for S2T data processing/model training
pip install pandas torchaudio sentencepiece

# En ASR
python examples/speech_to_text/prep_covost_data.py \
  --data-root ${COVOST_ROOT} --vocab-type char --src-lang en
# ST
python examples/speech_to_text/prep_covost_data.py \
  --data-root ${COVOST_ROOT} --vocab-type char \
  --src-lang fr --tgt-lang en
```

The generated files (manifest, features, vocabulary and data configuration) will be added to
`${COVOST_ROOT}/${SOURCE_LANG_ID}`.

Download our vocabulary files if you want to use our pre-trained models:

- ASR: [En](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_en_asr_vocab_char.zip)
- ST: [Fr-En](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_fr_en_st_vocab_char.zip)
  , [De-En](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_de_en_st_vocab_char.zip)
  , [Es-En](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_es_en_st_vocab_char.zip)
  , [Ca-En](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_ca_en_st_vocab_char.zip)
  , [En-De](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_en_de_st_vocab_char.zip)
  , [En-Ca](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_en_ca_st_vocab_char.zip)
  , [En-Fa](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_en_fa_st_vocab_char.zip)
  , [En-Et](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_en_et_st_vocab_char.zip)

## ASR

#### Training

We train an En ASR model for encoder pre-training some of the ST models.

```bash
fairseq-train ${COVOST_ROOT}/en \
  --config-yaml config_asr_en.yaml --train-subset train_asr_en --valid-subset dev_asr_en \
  --save-dir ${ASR_SAVE_DIR} --num-workers 4 --max-tokens 50000 --max-update 60000 \
  --task speech_to_text --criterion label_smoothed_cross_entropy --label-smoothing 0.1 \
  --report-accuracy --arch s2t_transformer_s --dropout 0.15 --optimizer adam --lr 2e-3 \
  --lr-scheduler inverse_sqrt --warmup-updates 10000 --clip-norm 10.0 --seed 1 --update-freq 8 \
  --attn-type None --pos-enc-type ${POS_ENC_TYPE}
```

where `ASR_SAVE_DIR` is the checkpoint root path and `POS_ENC_TYPE` refers to positional encoding to be used in the
conformer encoder. Set it to `abs`, `rope` or `rel_pos` to use the absolute positional encoding, rotary positional
encoding or relative positional encoding in the conformer layer respectively. Transformer encoder only supports absolute
positional encoding and by default, the transformer encoder will be used. To switch to conformer,
set `--attn-type espnet` and `--POS_ENC_TYPE`. We set `--update-freq 8` to simulate 8 GPUs with 1 GPU. You may want to
update it accordingly when using more than 1 GPU.

#### Inference & Evaluation

```bash
CHECKPOINT_FILENAME=avg_last_10_checkpoint.pt
python scripts/average_checkpoints.py \
  --inputs ${ASR_SAVE_DIR} --num-epoch-checkpoints 10 \
  --output "${ASR_SAVE_DIR}/${CHECKPOINT_FILENAME}"
fairseq-generate ${COVOST_ROOT}/en \
  --config-yaml config_asr_en.yaml --gen-subset test_asr_en --task speech_to_text \
  --path ${ASR_SAVE_DIR}/${CHECKPOINT_FILENAME} --max-tokens 50000 --beam 5 \
  --scoring wer --wer-tokenizer 13a --wer-lowercase --wer-remove-punct
```

#### Results

| --arch | --pos-enc-type | Params | En | Model |
|---|---|---|---|---|
| s2t_transformer_s | - | 31M | 25.6 | [Download](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_en_asr_transformer_s.pt) |
| s2t_conformer | rel_pos | 42.9M | 23.18| [Download](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_asr/rel_pos_asr_checkpoint_best.pt) |
| s2t_conformer | rope | 42.1M | 23.8| [Download](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_asr/rope_pos_asr_checkpoint_best.pt) |
| s2t_conformer | abs | 42.1M | 23.8| [Download](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_asr/abs_asr_checkpoint_best.pt) |

## ST

#### Training

Fr-En as example:

```bash
fairseq-train ${COVOST_ROOT}/fr \
  --config-yaml config_st_fr_en.yaml --train-subset train_st_fr_en --valid-subset dev_st_fr_en \
  --save-dir ${ST_SAVE_DIR} --num-workers 4 --max-update 30000 --max-tokens 40000 \  # --max-tokens 50000 for en-*
  --task speech_to_text --criterion label_smoothed_cross_entropy --label-smoothing 0.1 --report-accuracy \
  --arch s2t_transformer_s --encoder-freezing-updates 1000 --optimizer adam --lr 2e-3 \
  --lr-scheduler inverse_sqrt --warmup-updates 10000 --clip-norm 10.0 --seed 1 --update-freq 8 \
  --attn-type None --pos-enc-type ${POS_ENC_TYPE} \
  --load-pretrained-encoder-from ${ASR_SAVE_DIR}/${CHECKPOINT_FILENAME}
```

where `ST_SAVE_DIR` is the checkpoint root path and `POS_ENC_TYPE` refers to positional encoding to be used in the
conformer encoder. Set it to `abs`, `rope` or `rel_pos` to use the absolute positional encoding, rotary positional
encoding or relative positional encoding in the conformer layer respectively. Transformer encoder only supports absolute
positional encoding and by default, the transformer encoder will be used. To switch to conformer,
set `--attn-type espnet` and `--POS_ENC_TYPE`. Optionally load the pre-trained En ASR encoder for faster training and
better performance: `--load-pretrained-encoder-from <ASR checkpoint path>`. We set `--update-freq 8` to simulate 8 GPUs
with 1 GPU. You may want to update it accordingly when using more than 1 GPU.

#### Inference & Evaluation

Average the last 10 checkpoints and evaluate on test split:

```bash
CHECKPOINT_FILENAME=avg_last_10_checkpoint.pt
python scripts/average_checkpoints.py \
  --inputs ${ST_SAVE_DIR} --num-epoch-checkpoints 10 \
  --output "${ST_SAVE_DIR}/${CHECKPOINT_FILENAME}"
fairseq-generate ${COVOST_ROOT}/fr \
  --config-yaml config_st_fr_en.yaml --gen-subset test_st_fr_en --task speech_to_text \
  --path ${ST_SAVE_DIR}/${CHECKPOINT_FILENAME} \
  --max-tokens 50000 --beam 5 --scoring sacrebleu
```

## Interactive Decoding

Launch the interactive console via

```bash
fairseq-interactive ${COVOST_ROOT}/fr --config-yaml config_st_fr_en.yaml \
  --task speech_to_text --path ${SAVE_DIR}/${CHECKPOINT_FILENAME} \
  --max-tokens 50000 --beam 5
```

Type in WAV/FLAC/OGG audio paths (one per line) after the prompt.

#### Results

| --arch | --pos-enc-type | Params | ASR PT | Fr-En | De-En | Es-En | Ca-En | En-De | En-Ca | En-Fa | En-Et | Model |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| s2t_transformer | - | 31M | Yes | [27.2](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_fr_en_st_transformer_s.pt) | [17.7](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_de_en_st_transformer_s.pt) | [23.1](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_es_en_st_transformer_s.pt) | [19.3](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_ca_en_st_transformer_s.pt) | [16.1](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_en_de_st_transformer_s.pt) | [21.6](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_en_ca_st_transformer_s.pt) | [12.9](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_en_fa_st_transformer_s.pt) | [12.8](https://dl.fbaipublicfiles.com/fairseq/s2t/covost2_en_et_st_transformer_s.pt) | (<-Download) |
| s2t_conformer | rel_pos | 42.9M | No | [28.32](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/fr_en/rel_pos_from_scratch_avg_last_10_checkpoint.pt) | [18.21](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/de_en/rel_pos_from_scratch_avg_last_10_checkpoint.pt) | [25.98](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/es_en/rel_pos_from_scratch_avg_last_10_checkpoint.pt) | [21.13](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/ca_en/rel_pos_from_scratch_avg_last_10_checkpoint.pt) | [20.37](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_de/rel_pos_from_scratch_avg_last_10_checkpoint.pt) | [25.89](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_ca/rel_pos_from_scratch_avg_last_10_checkpoint.pt) | [15.59](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_fa/rel_pos_from_scratch_avg_last_10_checkpoint.pt) | [14.49](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_et/rel_pos_from_scratch_avg_last_10_checkpoint.pt) | (<-Download) |
| s2t_conformer | rel_pos | 42.9M | Yes| [27.15](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/fr_en/rel_pos_asr_pt_avg_last_10_checkpoint.pt) | [18.22](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/de_en/rel_pos_asr_pt_avg_last_10_checkpoint.pt) | [25.14](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/es_en/rel_pos_asr_pt_avg_last_10_checkpoint.pt) | [21.68](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/ca_en/rel_pos_asr_pt_avg_last_10_checkpoint.pt) | [20.35](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_de/rel_pos_asr_pt_avg_last_10_checkpoint.pt) | [25.92](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_ca/rel_pos_asr_pt_avg_last_10_checkpoint.pt) | [15.76](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_fa/rel_pos_asr_pt_avg_last_10_checkpoint.pt) | [16.52](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_et/rel_pos_asr_pt_avg_last_10_checkpoint.pt) | (<-Download) |
| s2t_conformer | rope | 42.1M | No | [27.61](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/fr_en/rope_from_scratch_avg_last_10_checkpoint.pt) | [17.6](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/de_en/rope_from_scratch_avg_last_10_checkpoint.pt) | [24.91](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/es_en/rope_from_scratch_avg_last_10_checkpoint.pt) | [20.78](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/ca_en/rope_from_scratch_avg_last_10_checkpoint.pt) | [19.7](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_de/rope_from_scratch_avg_last_10_checkpoint.pt) | [25.13](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_ca/rope_from_scratch_avg_last_10_checkpoint.pt) | [15.22](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_fa/rope_from_scratch_avg_last_10_checkpoint.pt) | [15.87](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_et/rope_from_scratch_avg_last_10_checkpoint.pt) | (<-Download) |
| s2t_conformer | rope | 42.1M | Yes | [26.99](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/fr_en/rope_asr_pt_avg_last_10_checkpoint.pt) | [17.71](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/de_en/rope_asr_pt_avg_last_10_checkpoint.pt) | [24.24](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/es_en/rope_asr_pt_avg_last_10_checkpoint.pt) | [21.24](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/ca_en/rope_asr_pt_avg_last_10_checkpoint.pt) | [19.9](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_de/rope_asr_pt_avg_last_10_checkpoint.pt) | [25.25](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_ca/rope_asr_pt_avg_last_10_checkpoint.pt) | [15.58](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_fa/rope_asr_pt_avg_last_10_checkpoint.pt) | [15.97](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_et/rope_asr_pt_avg_last_10_checkpoint.pt) | (<-Download) |
| s2t_conformer | abs | 42.1M | No | [27.45](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/fr_en/abs_from_scratch_avg_last_10_checkpoint.pt) | [17.25](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/de_en/abs_from_scratch_avg_last_10_checkpoint.pt) | [25.01](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/es_en/abs_from_scratch_avg_last_10_checkpoint.pt) |  [20.26](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/ca_en/abs_from_scratch_avg_last_10_checkpoint.pt) | [19.86](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_de/abs_from_scratch_avg_last_10_checkpoint.pt) | [25.25](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_ca/abs_from_scratch_avg_last_10_checkpoint.pt) | [15.46](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_fa/abs_from_scratch_avg_last_10_checkpoint.pt) | [15.81](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_et/abs_from_scratch_avg_last_10_checkpoint.pt) | (<-Download) |
| s2t_conforme | abs | 42.1M | Yes| [26.52](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/fr_en/abs_asr_pt_avg_last_10_checkpoint.pt) | [17.37](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/de_en/abs_asr_pt_avg_last_10_checkpoint.pt) | [25.40](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/es_en/abs_asr_pt_avg_last_10_checkpoint.pt) | [20.45](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/ca_en/abs_asr_pt_avg_last_10_checkpoint.pt) | [19.57](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_de/abs_asr_pt_avg_last_10_checkpoint.pt) | [25.40](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_ca/abs_asr_pt_avg_last_10_checkpoint.pt) | [15.17](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_fa/abs_asr_pt_avg_last_10_checkpoint.pt) | [15.83](https://dl.fbaipublicfiles.com/fairseq/conformer/covost2/en_et/abs_asr_pt_avg_last_10_checkpoint.pt) | (<-Download) |

[[Back]](..)
