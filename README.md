# Like a Good Nearest Neighbor: LaGoNN

Source code and data for [Like a Good Nearest Neighbor: Practical Content Moderation and Text Classification](https://arxiv.org/abs/2302.08957v3).

Contact person: Luke Bates, luke's_first_name.luke's_last_name@tu-darmstadt.de

https://www.ukp.tu-darmstadt.de/

https://www.tu-darmstadt.de/


Don't hesitate to send us an e-mail or report an issue, if something is broken (and it shouldn't be) or if you have further questions.

> This repository contains experimental software and is published for the sole purpose of giving additional background details on the respective publication.

## Project structure
* `lagonn.py` -- Like a Good Nearest Neighbor
* `get_data.py` -- script to download data files
* `main.py` -- code file that uses the other code files
* `modelling.py` -- code file for baselines
* `setup_utils.py` -- util code
* `train.py` -- code file for transformers
* `use_setfit.py` -- code file using and predicting with SetFit
* `custom_example.py` -- code file with examples for using LaGoNN for yourself.
* `dataframe_with_val/` -- data files
* `out_jsons/` -- where result jsons will be written
* `lagonn_both_examples/` -- examples from Appendix of LaGoNN BOTH output


## Requirements
Our results were computed in Python 3.9.13 with a 40 GB NVIDIA A100 Tensor Core GPU. Note that files will be written to disk if the code is run.

## Installation
To setup, please follow the instructions below.
```
git clone https://github.com/UKPLab/lagonn.git
cd lagonn
python -m venv mvenv
source mvenv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install torch==1.9.0+cu111 -f https://download.pytorch.org/whl/torch_stable.html
```
## Use LaGoNN in your own work
We find LAGONN_EXP to be the best mode when the training data is imbalanced. If the data is balanced, then we recommend using LAGONN. LABEL appears to be the more performant configuration of LaGoNN, but we encourage you to experiment with TEXT and BOTH. Please let us know if you see any interesting results.
Below, find a few examples of how to use LaGoNN. You can also look at `custom_example.py`.
#### LaGoNN_cheap
We use LaGoNN_cheap to modify our data before performing linear probing of the Sentence Transformer.

```python
from datasets import load_dataset
from lagonn import LaGoNN

#create huggingface dataset or load one
sst2 = load_dataset('SetFit/sst2') #for example, sst2 and sst5
sst2 = sst2.rename_column('label', 'labels')

sst5 = load_dataset('SetFit/sst5')
sst5 = sst5.rename_column('label', 'labels')
#the dataset needs to have a "text", "labels", and "label_text" field. We also assume a training, validation, and test split.

sst2_train_ds = sst2['train']
sst2_val_ds = sst2['validation']
sst2_test_ds = sst2['test']

sst5_train_ds = sst5['train']
sst5_val_ds = sst5['validation']
sst5_test_ds = sst5['test']


#We need to pass LaGoNN a dictionary of configurations. We used default SetFit settings in our experiments.
config_dict = {'lagonn_mode': 'LAGONN_CHEAP', # Don't finetune the embedding model 
               'lagonn_config': 'LABEL', # Use the gold label and Euclidean distance to modify input text
               'sample_size': 100, # How many examples per label to fine-tune the embedding model
               'st_model': 'paraphrase-mpnet-base-v2', # Choose your Sentence Transformer
               'batch_size': 16, # SetFit batch size
               'model_seed': 0, # Seed for training
               'metric': 'f1', # metric to pass to SetFit Trainer
               'num_iterations': 20, # The number of text pairs to generate for contrastive learning (see https://github.com/huggingface/setfit)
               'num_epochs': 1, # The number of epochs to use for contrastive learning (see https://github.com/huggingface/setfit)
               'sample_seed': 0} # Seed used to sample data

lgn = LaGoNN(train_ds=sst2_train_ds, val_ds=sst2_val_ds, test_ds=sst2_test_ds, config_dict=config_dict)
eval_dict = lgn.custom()
# If the dataset is binary, we compute the average precision, binary F1, and accuracy score.
print(eval_dict)
#example output:
#{'train_ap': 98.15036812378, 
#'train_f1_binary': 93.27056217114372, 
#'train_accuracy': 92.97687861271676, 
#'test_ap': 98.0644895038675, 
#'test_f1_binary': 92.52437703141928, 
#'test_accuracy': 92.42174629324546}


lgn = LaGoNN(train_ds=sst5_train_ds, val_ds=sst5_val_ds, test_ds=sst5_test_ds, config_dict=config_dict)
eval_dict = lgn.custom()
# If the dataset is multiclass, we compute the macro and micro F1 and the accuracy score.
print(eval_dict)
#example output:
# {'train_f1_macro': 59.86008435929301, 
# 'train_f1_micro': 61.73923220973783, 
# 'train_accuracy': 61.73923220973783, 
# 'test_f1_macro': 50.34429970479122, 
# 'test_f1_micro': 53.484162895927604, 
# 'test_accuracy': 53.484162895927604}
```
#### LaGoNN/LaGoNN_lite
Let's try fine-tuning the embedding model on a subset of the training data, for example, 100 examples per label (200 examples for sst-2, 500 examples for sst-5). We recommend this when you have a lot of balanced data.

```python
config_dict['lagonn_mode'] = 'LAGONN'
lgn = LaGoNN(train_ds=sst2_train_ds, val_ds=sst2_val_ds, test_ds=sst2_test_ds, config_dict=config_dict)
eval_dict = lgn.custom()
print(eval_dict)
#example output:
# {'train_ap': 96.92265641029665, 
# 'train_f1_binary': 92.19320415613592, 
# 'train_accuracy': 91.96531791907515, 
# 'test_ap': 97.26533880849458, 
# 'test_f1_binary': 93.1129476584022, 
# 'test_accuracy': 93.13563975837452}

lgn = LaGoNN(train_ds=sst5_train_ds, val_ds=sst5_val_ds, test_ds=sst5_test_ds, config_dict=config_dict)
eval_dict = lgn.custom()
print(eval_dict)
#example output:
# {'train_f1_macro': 53.83595113544369, 
# 'train_f1_micro': 54.260299625468164, 
# 'train_accuracy': 54.260299625468164, 
# 'test_f1_macro': 51.59342597162298, 
# 'test_f1_micro': 52.76018099547512, 
# 'test_accuracy': 52.76018099547512}
```
#### LaGoNN_exp
Finally, we can fine-tune the encoder on all of the training data. We recommend this when your data very imbalanced.

```python
config_dict['lagonn_mode'] = 'LAGONN_EXP'

lgn = LaGoNN(train_ds=sst2_train_ds, val_ds=sst2_val_ds, test_ds=sst2_test_ds, config_dict=config_dict)
eval_dict = lgn.custom()
print(eval_dict)
#example output:
# {'train_ap': 100.0, 
# 'train_f1_binary': 100.0, 
# 'train_accuracy': 100.0, 
# 'test_ap': 97.59521194883212, 
# 'test_f1_binary': 94.75958941112911, 
# 'test_accuracy': 94.67325645249862}


lgn = LaGoNN(train_ds=sst5_train_ds, val_ds=sst5_val_ds, test_ds=sst5_test_ds, config_dict=config_dict)
eval_dict = lgn.custom()
print(eval_dict)
#example output:
# {'train_f1_macro': 99.65678649502762, 
# 'train_f1_micro': 99.68398876404494, 
# 'train_accuracy': 99.68398876404494, 
# 'test_f1_macro': 55.28123100607829, 
# 'test_f1_micro': 56.289592760180994, 
# 'test_accuracy': 56.289592760180994}


```

### Reproduce our results
Our datafiles are too big for Github, but if you run
```
python get_data.py
```
then they will be downloaded and written to `dataframes_with_val/`.

#### A note on the data
The content moderation datasets need to be downloaded (see the README in data) and moved to a directory `dataframes_with_val/` in the same directory as `main.py`. The other datasets will be downloaded automatically from Huggingface. Note that `liar` refers to the collapsed version from the paper while `orig_liar` refers to the original dataset.

Then, you can run our code with `python main.py`. You can specifiy which configurations by passing arguments to python. 
* There are the following modes from the paper: 
    * PROBE (Sentence Transformer + logistic regression)
    * LOG_REG (SetFit in the paper)
    * KNN (kNN)
    * SETFIT_LITE (SetFit_lite)
    * SETFIT (SetFit_exp in the paper)
    * ROBERTA_FREEZE (RoBERTa_freeze)
    * ROBERTA_FULL (RoBERTa_full)
    * LAGONN_CHEAP (LaGoNN_cheap)
    * LAGONN (LaGoNN)
    * LAGONN_LITE (LaGoNN_lite) 
    * LAGONN_EXP (LaGoNN_exp)
* If you use a LaGoNN-based mode, you will also need to specific a LaGoNN Config:
    * ONLY_LABEL (LABEL in the paper)
    * DISTANCE
    * LABEL (LabDist in the paper)
    * TEXT
    * ALL

* You can pass any Sentence Transformer. We used paraphrase-mpnet-base-v2.
* You can pass any Transformer. We used roberta-base.
* You can pass any dataset we used as a 'task'. Note that we assume a `text`, `labels`, and `label_text` field and a training, validation, and test dataset.
* You can choose different distance precisions to round the Euclidean distance to with `--DISTANCE_PRECISION`.
* You can choose how many neighbors you would like to consider when modifying text with `--NUM_NEIGHBORS`. We tried 1 to 5.
* You can pass any seed for sampling data. We used 0, 1, 2, 3, and 4.
* You can turn on/off the file writer.


For example, if you wish to reproduce our LaGoNN_cheap results with seed = 0 on insincere-questions and write files to disk:
```
python main.py --ST_MODEL=paraphrase-mpnet-base-v2\
               --SEED=0\
               --TASK=insincere-questions\
               --MODE=LAGONN_EXP\
               --LAGONN_CONFIG=LABEL\
               --WRITE=True\
```
```
If you wish to reproduce our LaGoNN_exp results with seed = 4 on the original LIAR dataset, round to 5 decimals, modify with 5 neighbors and write files to disk:
```
python main.py --ST_MODEL=paraphrase-mpnet-base-v2\
               --SEED=0\
               --TASK=orig_liar\
               --MODE=LAGONN_EXP\
               --LAGONN_CONFIG=LABEL\
               --DISTANCE_PRECISION=5\
               --NUM_NEIGHBORS=5\
               --WRITE=True\
```
```
If you wish to reproduce our RoBERTa_full results on Toxic conversations with seed  = 3 and not write files to disk:
```
python main.py --TRANSFORMER_CLF=roberta-base\
               --SEED=3\
               --TASK=toxic_conversations\
               --MODE=ROBERTA_FULL\
```
If you wish to use LaGoNN_exp on Hate Speech Offensive with the BOTH config and write files to disk with seed = 2:
```
python main.py --ST_MODEL=paraphrase-mpnet-base-v2\
               --SEED=2\
               --TASK=hate_speech_offensive\
               --MODE=LAGONN_EXP\
               --LAGONN_CONFIG=BOTH\
               --WRITE=True\
```

### Expected results
Once finished, results will be written in the following format:
`out_jsons/{task}/{mode}/{balance}/{seed}/{step}/(num_neighbors)(-)(LaGoNN Config!)(dist_precision)(-)results.json`
Note that you will need to complete all five (0-4) seeds. This is because we report the average over the five seed for both the macro F1 and average precision.

### Citation
If our work was helpful for your work, please be so kind as to cite us:

```
@inproceedings{bates-gurevych-2024-like,
    title = "Like a Good Nearest Neighbor: Practical Content Moderation and Text Classification",
    author = "Bates, Luke  and
      Gurevych, Iryna",
    editor = "Graham, Yvette  and
      Purver, Matthew",
    booktitle = "Proceedings of the 18th Conference of the European Chapter of the Association for Computational Linguistics (Volume 1: Long Papers)",
    month = mar,
    year = "2024",
    address = "St. Julian{'}s, Malta",
    publisher = "Association for Computational Linguistics",
    url = "https://aclanthology.org/2024.eacl-long.17/",
    pages = "276--297",
    abstract = "Few-shot text classification systems have impressive capabilities but are infeasible to deploy and use reliably due to their dependence on prompting and billion-parameter language models. SetFit (Tunstall, 2022) is a recent, practical approach that fine-tunes a Sentence Transformer under a contrastive learning paradigm and achieves similar results to more unwieldy systems. Inexpensive text classification is important for addressing the problem of domain drift in all classification tasks, and especially in detecting harmful content, which plagues social media platforms. Here, we propose Like a Good Nearest Neighbor (LaGoNN), a modification to SetFit that introduces no learnable parameters but alters input text with information from its nearest neighbor, for example, the label and text, in the training data, making novel data appear similar to an instance on which the model was optimized. LaGoNN is effective at flagging undesirable content and text classification, and improves SetFit`s performance. To demonstrate LaGoNN`s value, we conduct a thorough study of text classification systems in the context of content moderation under four label distributions, and in general and multilingual classification settings."
}
```
