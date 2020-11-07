# FAST AND ACCURATE READING COMPREHENSION BY COMBINING SELF-ATTENTION AND CONVOLUTION
A Tensorflow implementation of Google's [Fast Reading Comprehension (FRC)](https://openreview.net/pdf?id=B14TlG-RW) from [ICLR2018](https://openreview.net/forum?id=B14TlG-RW).
Training and preprocessing pipeline has been adopted from [R-Net by HKUST-KnowComp](https://github.com/HKUST-KnowComp/R-Net). Demo mode is working. After training, just use `python config.py --mode demo` to run an interactive demo server.

Due to memory issue, a single head dot-product attention is used as opposed to 8 heads multi-head attention as mentioned in the original paper. Also hidden size is reduced to 96 from 128 due to using GTX1080 compared to p100 in the paper. (8GB GPU memory is insufficient. If you have a 12GB memory GPU please share your train results with us.)

The current best model reaches EM/F1 = 70.0/79.4 in 60k steps (about 6 hours with bucket / 8 hours without bucket in GTX1080). Detailed results are listed below.

![Alt text](/../master/screenshots/figure.png?raw=true "Network Outline")

## Dataset
The dataset used for this task is [Stanford Question Answering Dataset](https://rajpurkar.github.io/SQuAD-explorer/).
Pretrained [GloVe embeddings](https://nlp.stanford.edu/projects/glove/) obtained from common crawl with 840B tokens are used for words.

## Requirements
  * Python>=2.7
  * NumPy
  * tqdm
  * TensorFlow>=1.5
  * spacy==2.0.9 (only if you want to load the [pretrained model](https://drive.google.com/open?id=1gJtcPBNuDr9_2LuP_4x_4VN6_5fQCdfB), otherwise lower versions are fine)
  * bottle (only for demo)

## Usage
To download and preprocess the data, run

```bash
# download SQuAD and Glove
sh download.sh
# preprocess the data
python config.py --mode prepro
```

Just like [R-Net by HKUST-KnowComp](https://github.com/HKUST-KnowComp/R-Net), hyper parameters are stored in config.py. For debug/train/test/demo, run

```bash
python config.py --mode debug/train/test/demo
```

To evaluate the model with official code, run
```bash
python evaluate-v1.1.py ~/data/squad/dev-v1.1.json train/{model_name}/answer/answer.json
```

The default directory for tensorboard log file is `train/{model_name}/event`

### Pretrained Model
Pretrained model weights are available [here](https://drive.google.com/open?id=1gJtcPBNuDr9_2LuP_4x_4VN6_5fQCdfB). Download and unpack "FRC" folder in the "train" directory.

## Detailed Implementaion

  * The model adopts character level convolution - max pooling - highway network for input representations similar to [this paper by Yoon Kim](https://arxiv.org/pdf/1508.06615.pdf).
  * Encoder consists of positional encoding - depthwise separable convolution - self attention - feed forward structure with layer norm in between.
  * Despite the original paper using 200, we observe that smaller character dimension lead to better generalization.
  * For regularization, dropout of 0.1 is used every 2 sub-layers and 2 blocks.
  * Stochastic depth dropout is used to drop the residual connection with respect to increasing depth of the network as this model heavily relies on residual connections.
  * Query-to-Context attention is used along with Context-to-Query attention, which seems to improve the performance more than what the paper reported. This may be due to the lack of diversity in self attention due to 1 head (as opposed to 8 heads) which may have repetitive information that query-to-context attention contains.
  * Learning rate increases from 0.0 to 0.001 in first 1000 steps in inverse exponential scale and fixed to 0.001 from 1000 steps.
  * At inference, this model uses shadow variables maintained by exponential moving average of all global variables.
  * This model uses training / testing / preprocessing pipeline from [R-Net](https://github.com/HKUST-KnowComp/R-Net) for better efficiency.

## Results
Here is the collected results from this repository and the original paper.

|      Model     | Training Steps | Size | Attention Heads | Data Size (aug) |  EM  |  F1  |
|:--------------:|:--------------:|:----:|:---------------:|:---------------:|:----:|:----:|
|       My Model |     35,000     |  96  |        1        |   87k (no aug)  | 69.0 | 78.0 |
|       My model |     60,000     |  96  |        1        |   87k (no aug)  | 70.0 | 79.4 |
| Original Paper |     35,000     |  128 |        8        |   87k (no aug)  |  NA  | 77.0 |
| Original Paper |     150,000    |  128 |        8        |   87k (no aug)  | 72.5 | 81.4 |
| Original Paper |     340,000    |  128 |        8        |    240k (aug)   | 76.2 | 84.6 |

## TODO's
- [x] Training and testing the model
- [x] Add trilinear function to Context-to-Query attention
- [x] Apply dropouts + stochastic depth dropout
- [x] Query-to-context attention
- [x] Realtime Demo
- [ ] Data augmentation by paraphrasing
- [ ] Train with full hyperparameters (Augmented data, 8 heads, hidden units = 128)

## Tensorboard
Run tensorboard for visualisation.
```shell
$ tensorboard --logdir=./
```
