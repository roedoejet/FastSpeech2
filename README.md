# FastSpeech 2 - PyTorch Implementation

This fork of ming024's implementation was used for my M.Sc. Speech & Language Processing dissertation, "Low Resource Speech Synthesis". The differences are outlined below.

If you are looking for checkpoints for the LJ models discussed in our [ACL2022 paper](https://aclanthology.org/2022.acl-long.507/), please visit this repo: https://github.com/roedoejet/FastSpeech2_ACL2022_reproducibility (unfortunately GitHub doesn't allow Git LFS on forks, which is why I was forced to use a separate repo). If you use this or do more research I'd be really excited to hear about it, so please [cite us](#citation) and [get in touch](https://aidanpine.ca)! 

# Changes from ming024
- Model allows removal of energy variance adaptor, postnet and optional depthwise separable convolutions similar to LightSpeech (Luo et. al. 2021). Changes can be made in `config/*/model.yaml`
- Model allows for one-hot learnable language embeddings. Changes can be made in `config/*/model.yaml`
- Model allows for deep-speaker based speaker embeddings (in addition to one-hot implementation from ming024). Clone with `--recurse-submodule` or run `git submodule update --init` after cloning to enable. Changes can be made in `config/*/model.yaml`
- Inputs can be multihot phonological feature vectors instead of one-hot character/phone embeddings. See Gutkin et. al. 2018, Wells and Richmond 2021, and my dissertation. For this to work you must have a mapping to IPA. Panphon is used to generate 24 segmental features. If your language uses phonemic tone, please amend features.py to use the 7 features from Wang 1967. Otherwise set the number of features accordingly in `config/*/model.py` and `config/*/preprocess.py`.

This is a PyTorch implementation of Microsoft's text-to-speech system [**FastSpeech 2: Fast and High-Quality End-to-End Text to Speech**](https://arxiv.org/abs/2006.04558v1). 
This project is based on [xcmyz's implementation](https://github.com/xcmyz/FastSpeech) of FastSpeech. Feel free to use/modify the code.

There are several versions of FastSpeech 2.
This implementation is more similar to [version 1](https://arxiv.org/abs/2006.04558v1), which uses F0 values as the pitch features.
On the other hand, pitch spectrograms extracted by continuous wavelet transform are used as the pitch features in the [later versions](https://arxiv.org/abs/2006.04558).

![](./img/model.png)

# Updates
- 2021/7/8: Release the checkpoint and audio samples of a multi-speaker English TTS model trained on LibriTTS
- 2021/2/26: Support English and Mandarin TTS
- 2021/2/26: Support multi-speaker TTS (AISHELL-3 and LibriTTS)
- 2021/2/26: Support MelGAN and HiFi-GAN vocoder

# Audio Samples
Audio samples generated by this implementation can be found [here](https://ming024.github.io/FastSpeech2/). 

# Quickstart

## Dependencies
I recommend using Conda and Python 3.7. To do that, create a new environment:

```
conda create --name FastSpeech2 python=3.7
conda activate FastSpeech2
```

You can then install the Python dependencies with
```
pip3 install -r requirements.txt
```

## Inference

You have to download the [pretrained models](https://drive.google.com/drive/folders/1DOhZGlTLMbbAAFZmZGDdc77kz1PloS7F?usp=sharing) and put them in ``output/ckpt/LJSpeech/``,  ``output/ckpt/AISHELL3``, or ``output/ckpt/LibriTTS/``.
Checkpoints should be named according to the number of training steps, so you will need to remove the dataset name from these pretrained model files.
You will also need to unzip the HiFi-GAN checkpoint specified in the corresponding `model.yaml` config file, e.g. `cd hifigan && unzip generator_universal.pth.tar.zip`.
Finally, uncomment the noted lines in `text/symbols.py` to export the correct list of `SYMBOLS` for these pre-trained models.

For English single-speaker TTS, run
```
python3 synthesize.py --text "YOUR_DESIRED_TEXT" --restore_step 900000 --mode single -p config/LJSpeech/preprocess.yaml -m config/LJSpeech/model.yaml -t config/LJSpeech/train.yaml
```

For Mandarin multi-speaker TTS, try
```
python3 synthesize.py --text "大家好" --speaker_id SPEAKER_ID --restore_step 600000 --mode single -p config/AISHELL3/preprocess.yaml -m config/AISHELL3/model.yaml -t config/AISHELL3/train.yaml
```

For English multi-speaker TTS, run
```
python3 synthesize.py --text "YOUR_DESIRED_TEXT"  --speaker_id SPEAKER_ID --restore_step 800000 --mode single -p config/LibriTTS/preprocess.yaml -m config/LibriTTS/model.yaml -t config/LibriTTS/train.yaml
```

The generated utterances will be put in ``output/result/``.

Here is an example of synthesized mel-spectrogram of the sentence "Printing, in the only sense with which we are at present concerned, differs from most if not from all the arts and crafts represented in the Exhibition", with the English single-speaker TTS model.  
![](./img/synthesized_melspectrogram.png)

## Batch Inference
Batch inference is also supported, try

```
python3 synthesize.py --source val.txt --restore_step 900000 --mode batch -p config/LJSpeech/preprocess.yaml -m config/LJSpeech/model.yaml -t config/LJSpeech/train.yaml
```
to synthesize all utterances in ``preprocessed_data/LJSpeech/val.txt`` (where the path to `val.txt` is specified in `config/LJSpeech/preprocess.yaml` as `preprocessed_path`).

## Controllability
The pitch/volume/speaking rate of the synthesized utterances can be controlled by specifying the desired pitch/energy/duration ratios.
For example, one can increase the speaking rate by 20 % and decrease the volume by 20 % by

```
python3 synthesize.py --text "YOUR_DESIRED_TEXT" --restore_step 900000 --mode single -p config/LJSpeech/preprocess.yaml -m config/LJSpeech/model.yaml -t config/LJSpeech/train.yaml --duration_control 0.8 --energy_control 0.8
```

# Training

## Datasets

The supported datasets are

- [LJSpeech](https://keithito.com/LJ-Speech-Dataset/): a single-speaker English dataset consists of 13100 short audio clips of a female speaker reading passages from 7 non-fiction books, approximately 24 hours in total.
- [AISHELL-3](http://www.aishelltech.com/aishell_3): a Mandarin TTS dataset with 218 male and female speakers, roughly 85 hours in total.
- [LibriTTS](https://research.google/tools/datasets/libri-tts/): a multi-speaker English dataset containing 585 hours of speech by 2456 speakers.

We take LJSpeech as an example hereafter.

## Preprocessing
 
First, run 
```
python3 prepare_align.py config/LJSpeech/preprocess.yaml
```
for some preparations.

As described in the paper, [Montreal Forced Aligner](https://montreal-forced-aligner.readthedocs.io/en/latest/) (MFA) is used to obtain the alignments between the utterances and the phoneme sequences.
Alignments of the supported datasets are provided [here](https://drive.google.com/drive/folders/1DBRkALpPd6FL9gjHMmMEdHODmkgNIIK4?usp=sharing).
You have to unzip the files in ``preprocessed_data/LANGUAGE_CODE/SPEAKER_ID/TextGrid/``, where `LANGUAGE_CODE` is specified in your `preprocess.yaml` under `preprocessing.text.language` and `SPEAKER_ID` is determined by the corpus-specific preprocessor defined in `preprocessor/yourcorpus.py` (e.g. `preprocessed_data/eng/LJSpeech/TextGrid`).

After that, run the preprocessing script by
```
python3 preprocess.py config/LJSpeech/preprocess.yaml
```

Alternately, you can align the corpus by yourself. 
Download the official MFA package and run
```
./montreal-forced-aligner/bin/mfa_align raw_data/LJSpeech/ lexicon/librispeech-lexicon.txt english preprocessed_data/LJSpeech
```
or
```
./montreal-forced-aligner/bin/mfa_train_and_align raw_data/LJSpeech/ lexicon/librispeech-lexicon.txt preprocessed_data/LJSpeech
```

to align the corpus and then run the preprocessing script.
```
python3 preprocess.py config/LJSpeech/preprocess.yaml
```

## Training

Train your model with
```
python3 train.py -p config/LJSpeech/preprocess.yaml -m config/LJSpeech/model.yaml -t config/LJSpeech/train.yaml
```

The model takes less than 10k steps (less than 1 hour on my GTX1080Ti GPU) of training to generate audio samples with acceptable quality, which is much more efficient than the autoregressive models such as Tacotron2.

# TensorBoard

Use
```
tensorboard --logdir output/log/LJSpeech
```

to serve TensorBoard on your localhost.
The loss curves, synthesized mel-spectrograms, and audios are shown.

![](./img/tensorboard_loss.png)
![](./img/tensorboard_spec.png)
![](./img/tensorboard_audio.png)

# Adding a new language

- update the config in `config/YourLanguage`. Minimally change all the values with "change this" comments
- add your input symbols to `text/symbols.py`
- add your cleaner to `text/cleaners.py`
- add your language specific preprocessors to `preprocessor/yourlanguage.py` and `synthesize.py`
- run MFA on your data and add to `preprocessed_data/YourLanguage/TextGrid`
- create lexicon for your data and add to `lexicon/`
- preprocess your data with `python3 preprocess.py config/YourLanguage/preprocess.yaml`
- train your system with `python3 train -p config/YourLanguage/preprocess.yaml -m config/YourLanguage/model.yaml -t config/YourLanguage/train.yaml`
- synthesize speech with `python3 synthesize.py --text "YOUR_DESIRED_TEXT" --restore_step 300000 --mode single -p config/YourLanguage/preprocess.yaml -m config/YourLanguage/model.yaml -t config/YourLanguage/train.yaml`

# Implementation Issues

- Following [xcmyz's implementation](https://github.com/xcmyz/FastSpeech), I use an additional Tacotron-2-styled Post-Net after the decoder, which is not used in the original FastSpeech 2.
- Gradient clipping is used in the training.
- In my experience, using phoneme-level pitch and energy prediction instead of frame-level prediction results in much better prosody, and normalizing the pitch and energy features also helps. Please refer to ``config/README.md`` for more details.

Please inform me if you find any mistakes in this repo, or any useful tips to train the FastSpeech 2 model.

# References
- [FastSpeech 2: Fast and High-Quality End-to-End Text to Speech](https://arxiv.org/abs/2006.04558), Y. Ren, *et al*.
- [xcmyz's FastSpeech implementation](https://github.com/xcmyz/FastSpeech)
- [TensorSpeech's FastSpeech 2 implementation](https://github.com/TensorSpeech/TensorflowTTS)
- [rishikksh20's FastSpeech 2 implementation](https://github.com/rishikksh20/FastSpeech2)

# Citation

Please cite [the paper](https://arxiv.org/abs/2006.04558). The citations for this implementation are as follows:

Original Implementation:
```
@INPROCEEDINGS{chien2021investigating,
  author={Chien, Chung-Ming and Lin, Jheng-Hao and Huang, Chien-yu and Hsu, Po-chun and Lee, Hung-yi},
  booktitle={ICASSP 2021 - 2021 IEEE International Conference on Acoustics, Speech and Signal Processing (ICASSP)}, 
  title={Investigating on Incorporating Pretrained and Learnable Speaker Representations for Multi-Speaker Multi-Style Text-to-Speech}, 
  year={2021},
  volume={},
  number={},
  pages={8588-8592},
  doi={10.1109/ICASSP39728.2021.9413880}}
```

This fork:
```
@inproceedings{pine-etal-2022-requirements,
    title = "Requirements and Motivations of Low-Resource Speech Synthesis for Language Revitalization",
    author = "Pine, Aidan  and
      Wells, Dan  and
      Brinklow, Nathan  and
      Littell, Patrick  and
      Richmond, Korin",
    booktitle = "Proceedings of the 60th Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)",
    month = may,
    year = "2022",
    address = "Dublin, Ireland",
    publisher = "Association for Computational Linguistics",
    url = "https://aclanthology.org/2022.acl-long.507",
    doi = "10.18653/v1/2022.acl-long.507",
    pages = "7346--7359",
    abstract = "This paper describes the motivation and development of speech synthesis systems for the purposes of language revitalization. By building speech synthesis systems for three Indigenous languages spoken in Canada, Kanien{'}k{\'e}ha, Gitksan {\&} SEN{\'C}OŦEN, we re-evaluate the question of how much data is required to build low-resource speech synthesis systems featuring state-of-the-art neural models. For example, preliminary results with English data show that a FastSpeech2 model trained with 1 hour of training data can produce speech with comparable naturalness to a Tacotron2 model trained with 10 hours of data. Finally, we motivate future research in evaluation and classroom integration in the field of speech synthesis for language revitalization.",
}
```
