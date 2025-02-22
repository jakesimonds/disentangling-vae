# Disentagled VAE -- Jake's Colab .ipynb Implementation

This is a fork of YannDubs/disentangling-vae--great repo, 10/10 would recommend. This fork really doesn't add much, I just wanted to **1)** 
document my Colab implementation that as of 6/15/23 works reasonably well (though I did pay for $10/month colab, not sure how it might do on the free tier) and **2)** 
document my own results from training the model on the CelebA dataset, and spending some time with the paper Disentangling B-VAE (https://arxiv.org/abs/1804.03599).

# Colab Quickstart

### Step 1:

Either go to this link (https://colab.research.google.com/drive/10c68VNn7E673TKLLS-G2FS6WBrBnCVkD?usp=sharing) or download and then open in Google Colab the file "Disentangling_Colab.ipynb" (might need to make a copy to be able to make changes).

### Step 2: 

Follow instructions in notebook! 

Basically I have it set up so you can just name a folder where the results will save, set your hyper-parameters (following instructions from original repo), and then run it. If runtime stays uninterrupted (can be a little dicey with colab) your results will save automatically to your google drive. 

# My Findings

##  B - VAE successfully factorized the CelebA dataset into meaningful latent varariables

Borrowing some hyperparameters from the original disentangling-vae github my first interesting result was that Just as the paper promised, B-VAE could factorize the CelebA dataset into interesting latent dimensions.

### Manually adjusting number of latent variables, z

```commandline
!python /content/disentangling-vae/main.py {FOLDER_NAME} -d celeba -l betaB --lr 0.0005 -b 256 -e 150
```
<img src="Colab_Implementation_Files/figs/training_first_run.png" width=550>

Though some of the latent traversals (looking across the ten rows right-to-left) are not immediately intelligible, some are! 

Also of note I think is that we can see relatively smooth transitions along the latent traversals.  

### Number of latent channels reduced 50% (from 10 (default) to 5)

```commandline
!python /content/disentangling-vae/main.py {FOLDER_NAME} -d celeba -l betaB --lr 0.0005 -b 256 -z 5 -e 150
```

<img src="Colab_Implementation_Files/figs/training_second.png" width=550>

With the number of latent channels cut in half, we no longer see smooth transition. Not only is each channel encoding multiple elements that a human would call seperate generative factors, but these generative factors are showing up in multiple channels! We can see "blue background" being encoded in four of the five channels, for example.  

### 30 latent channels

<img src="Colab_Implementation_Files/figs/training_30z.png" width=300>

```commandline
!python /content/disentangling-vae/main.py {FOLDER_NAME} -d celeba -l betaB --betaB-finC 100 --lr 0.0005 -b 256 -z 30 -e 30
```

Naively one might think that more open channels would mean more information could be passed through and reconstruction would improve. While one could potentially aim to set up experimental conditions where that takes place, under the experimental conditions I worked with adding more channels just resulted in a number of channels lying dormant. 

The equation determining encoding has a reconstruction term and a penalty term. Here I believe we are seeing the penalty term "put its foot down" and not allowing channels to encode information that would result in an increase in the KL divergence to a magnitiude that offsets the benefit to the reconstruction term.  


##  Reconstruction: Not great, but insights/intuition in how it executed

<img src="Colab_Implementation_Files/figs/reconstruct.png" width=550>

This is the reconstruction from the experiment 1 above (with 10 latent channels). Top three rows are the input images, bottom three rows are the corresponding reconstructions. 

 My takeaway from these reconstructions was it pointed out things about the celebA dataset that I (having never done this before) really hadn't thought about too much: just how much variation there is in the background, lighting, pose, etc. 

The reconstruction does an okay job on some of those elements, while the faces themselves are not at least to my eye very faithful. 

Also of note: 
- it seems to reconstruct more poorly on non-white faces, likely an artifact of the dataset population
- Things like hats or glasses are often missing(see Ryan Braun, second row, second from the left, reconstructed with decent scale, background color & pose but no hat & a pretty generic face). It would be very cool to tease out latent dimensions for items like those, and I think theoretically it's possible. But to get to that level of granularity, though, you'd probably need to go with a different approach in both data and framework. The CelebA dataset might not be ideal because you've just got a lot of variation in the images that will "count for more" in reconstruction fidelity than something like a hat-vs-no-hat or glasses-vs-no-glasses. And the algorithm would probably also take a heck of a lot of fine tuning to both open up but then also make factorized use of the more latent channels you'd need to start getting to some of those more granular things. 


## Recreating Figure 2 from "Understanding Disentanglement in B-VAE" with Celeba dataset

<img src="Colab_Implementation_Files/figs/paper_fig2.png" width=550>

Figure 2 from "Understanding Disentangling in B-VAE" (reproduced above) illustrates how a higher beta value leads to a less entangled latent space. They mention in the paper that when the beta value is high, a potential drawback is loss of reconstruction fidelity, but we don't see that in figure 2 because two latent information channels are sufficient to reproduce the Gaussian blobs.

<div style="display: flex;">
    <img src="Colab_Implementation_Files/figs/reconstruct_b1.png" width="400">
    <img src="Colab_Implementation_Files/figs/training_b1.png" width="400">
</div>

Beta = 1

```commandline
!python /content/disentangling-vae/main.py {FOLDER_NAME} -d celeba -l betaH --betaH-B 1 --lr 0.0005 -b 256 -e 60
```

With Beta=1 not heavily penalizing the regularization term we see that all our latent channels are in use, but they are not factorized (compare to experiment 1 above).


<div style="display: flex;">
<img src="Colab_Implementation_Files/figs/reconstruct_150.png" width=400>
<img src="Colab_Implementation_Files/figs/training_150.png" width=400>
</div>
Beta = 150

```commandline
!python /content/disentangling-vae/main.py {FOLDER_NAME} -d celeba -l betaH --betaH-B 150 --lr 0.0005 -b 256 -e 60
```

With Beta=150 heavily penalizing the regularization term, only three channels are in use, and they are factorized. Reconstruction fidelity, however, has taken a pretty significant hit.

## It is hard to know how long to train for ( # of epochs specifically, but all hyperparameters really) 

The careful reader may have noticed that in the experiments I show the # of epochs varies quite widely. 

<img src="Colab_Implementation_Files/figs/June15_run_loss.png" width=350>

This is my first time doing something like this, and I really didn't know where to begin for setting hyperparameters. The above graph shows average loss over total training time for a run with 150 epochs, and after seeing it laid out like this I started doing shorter trainings since my ambition here is to build intuition and explore rather than robustly prove anything. Since 150 epoch runs were taking in the ballpark of 6 hours to complete, I wish I had made this graph earlier! 

## Next Steps

It might be interesting to train with a slightly "cleaner" dataset of faces. The Olivetti Faces dataset was something I considered, and I think would be perfect in terms of having a lower resolution (so training could be quicker) and more consistent in extraneous factors, so maybe a model trained on a dataset like that would have more interesting factors to tease out since it wouldn't have to encode as much info about variation in background, lighting, pose, etc. Unfortunately Olivetti is a very small dataset, with only 400 total images. But I might still try it, just ran out of time for now. 

## Why Colab? 

I had access to the Discovery HPC cluster for this project, but implemented in Colab. The main reason for this is that I was more familiar with Colab, and very quickly was able to get a "working" clone of the repo, and then when I tried to reproduce that clone on the Discovery cluster I ran into some dependency issues. 

I'm not sure how serious those dependency issues were because honestly I didn't spend much time troubleshooting. I talked myself into Colab by recalling some youtube videos I've watched where people have done much more complicated things than what I attempted here using colab, so I felt reasonably confident it would do what I needed. I also thought that since I'll be losing access to the discovery cluster after this course, maybe it would be pragmatic to learn a tool I can keep using. Now on the other side of things, I do kind of wish I had worked through the dependency issues. I just was a bit intimidated by an unfamiliar environment. 

# Disentangled VAE [![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://github.com/YannDubs/disentangling-vae/blob/master/LICENSE) [![Python 3.6+](https://img.shields.io/badge/python-3.6+-blue.svg)](https://www.python.org/downloads/release/python-360/)

This repository contains code (training / metrics / plotting) to investigate disentangling in VAE as well as compare 5 different losses ([summary of the differences](#losses-explanation)) using a [single architecture](#single-model-comparison):

* **Standard VAE Loss** from [Auto-Encoding Variational Bayes](https://arxiv.org/abs/1312.6114)
* **β-VAE<sub>H</sub>** from [β-VAE: Learning Basic Visual Concepts with a Constrained Variational Framework](https://openreview.net/pdf?id=Sy2fzU9gl)
* **β-VAE<sub>B</sub>** from [Understanding disentangling in β-VAE](https://arxiv.org/abs/1804.03599)
* **FactorVAE** from [Disentangling by Factorising](https://arxiv.org/abs/1802.05983)
* **β-TCVAE** from [Isolating Sources of Disentanglement in Variational Autoencoders](https://arxiv.org/abs/1802.04942)

Notes:
- Tested for python >= 3.6
- Tested for CPU and GPU

Table of Contents:
1. [Install](#install)
2. [Run](#run)
3. [Plot](#plot)
3. [Data](#data)
4. [Our Contributions](#our-contributions)
5. [Losses Explanation](#losses-explanation)
6. [Citing](#cite)

## Install

```
# clone repo
pip install -r requirements.txt
```

## Run

Use `python main.py <model-name> <param>` to train and/or evaluate a model. For example:

```
python main.py btcvae_celeba_mini -d celeba -l btcvae --lr 0.001 -b 256 -e 5
```

You can run predefined experiments and hyper-parameters using `-x <experiment>`. Those hyperparameters are found in `hyperparam.ini`. Pretrained models for each experiment can be found in `results/<experiment>` (created using `./bin/train_all.sh`).


### Output
This will create a directory `results/<saving-name>/` which will contain:

* **model.pt**: The model at the end of training. 
* **model-**`i`**.pt**: Model checkpoint after `i` iterations. By default saves every 10.
* **specs.json**: The parameters used to run the program (default and modified with CLI).
* **training.gif**: GIF of latent traversals of the latent dimensions Z at every epoch of training.
* **train_losses.log**: All (sub-)losses computed during training.
* **test_losses.log**: All (sub-)losses computed at the end of training with the model in evaluate mode (no sampling). 
* **metrics.log**: [Mutual Information Gap](https://arxiv.org/abs/1802.04942) metric and [Axis Alignment Metric](#axis-alignment-metric). Only if `--is-metric` (slow).


### Help
```
usage: main.py ...

PyTorch implementation and evaluation of disentangled Variational AutoEncoders
and metrics.

optional arguments:
  -h, --help            show this help message and exit

General options:
  name                  Name of the model for storing or loading purposes.
  -L, --log-level {CRITICAL,ERROR,WARNING,INFO,DEBUG,NOTSET}
                        Logging levels. (default: info)
  --no-progress-bar     Disables progress bar. (default: False)
  --no-cuda             Disables CUDA training, even when have one. (default:
                        False)
  -s, --seed SEED       Random seed. Can be `None` for stochastic behavior.
                        (default: 1234)

Training specific options:
  --checkpoint-every CHECKPOINT_EVERY
                        Save a checkpoint of the trained model every n epoch.
                        (default: 30)
  -d, --dataset {mnist,fashion,dsprites,celeba,chairs}
                        Path to training data. (default: mnist)
  -x, --experiment {custom,debug,best_celeba,VAE_mnist,VAE_fashion,VAE_dsprites,VAE_celeba,VAE_chairs,betaH_mnist,betaH_fashion,betaH_dsprites,betaH_celeba,betaH_chairs,betaB_mnist,betaB_fashion,betaB_dsprites,betaB_celeba,betaB_chairs,factor_mnist,factor_fashion,factor_dsprites,factor_celeba,factor_chairs,btcvae_mnist,btcvae_fashion,btcvae_dsprites,btcvae_celeba,btcvae_chairs}
                        Predefined experiments to run. If not `custom` this
                        will overwrite some other arguments. (default: custom)
  -e, --epochs EPOCHS   Maximum number of epochs to run for. (default: 100)
  -b, --batch-size BATCH_SIZE
                        Batch size for training. (default: 64)
  --lr LR               Learning rate. (default: 0.0005)

Model specfic options:
  -m, --model-type {Burgess}
                        Type of encoder and decoder to use. (default: Burgess)
  -z, --latent-dim LATENT_DIM
                        Dimension of the latent variable. (default: 10)
  -l, --loss {VAE,betaH,betaB,factor,btcvae}
                        Type of VAE loss function to use. (default: betaB)
  -r, --rec-dist {bernoulli,laplace,gaussian}
                        Form of the likelihood ot use for each pixel.
                        (default: bernoulli)
  -a, --reg-anneal REG_ANNEAL
                        Number of annealing steps where gradually adding the
                        regularisation. What is annealed is specific to each
                        loss. (default: 0)

BetaH specific parameters:
  --betaH-B BETAH_B     Weight of the KL (beta in the paper). (default: 4)

BetaB specific parameters:
  --betaB-initC BETAB_INITC
                        Starting annealed capacity. (default: 0)
  --betaB-finC BETAB_FINC
                        Final annealed capacity. (default: 25)
  --betaB-G BETAB_G     Weight of the KL divergence term (gamma in the paper).
                        (default: 1000)

factor VAE specific parameters:
  --factor-G FACTOR_G   Weight of the TC term (gamma in the paper). (default:
                        6)
  --lr-disc LR_DISC     Learning rate of the discriminator. (default: 5e-05)

beta-tcvae specific parameters:
  --btcvae-A BTCVAE_A   Weight of the MI term (alpha in the paper). (default:
                        1)
  --btcvae-G BTCVAE_G   Weight of the dim-wise KL term (gamma in the paper).
                        (default: 1)
  --btcvae-B BTCVAE_B   Weight of the TC term (beta in the paper). (default:
                        6)

Evaluation specific options:
  --is-eval-only        Whether to only evaluate using precomputed model
                        `name`. (default: False)
  --is-metrics          Whether to compute the disentangled metrcics.
                        Currently only possible with `dsprites` as it is the
                        only dataset with known true factors of variations.
                        (default: False)
  --no-test             Whether not to compute the test losses.` (default:
                        False)
  --eval-batchsize EVAL_BATCHSIZE
                        Batch size for evaluation. (default: 1000)
```

## Plot

Use `python main_viz.py <model-name> <plot_types> <param>` to plot using pretrained models. For example:

```
python main_viz.py btcvae_celeba_mini gif-traversals reconstruct-
                        traverse -c 7 -r 6 -t 2 --is-posterior
```

This will save the plots in the model directory  `results/<model-name>/`. Generated plots for all experiments are found in their respective directories (created using `./bin/plot_all.sh`).

### Help
```
usage: main_viz.py ...

CLI for plotting using pretrained models of `disvae`

positional arguments:
  name                  Name of the model for storing and loading purposes.
  {generate-samples,data-samples,reconstruct,traversals,reconstruct-traverse,gif-traversals,all}
                        List of all plots to generate. `generate-samples`:
                        random decoded samples. `data-samples` samples from
                        the dataset. `reconstruct` first rnows//2 will be the
                        original and rest will be the corresponding
                        reconstructions. `traversals` traverses the most
                        important rnows dimensions with ncols different
                        samples from the prior or posterior. `reconstruct-
                        traverse` first row for original, second are
                        reconstructions, rest are traversals. `gif-traversals`
                        grid of gifs where rows are latent dimensions, columns
                        are examples, each gif shows posterior traversals.
                        `all` runs every plot.

optional arguments:
  -h, --help            show this help message and exit
  -s, --seed SEED       Random seed. Can be `None` for stochastic behavior.
                        (default: None)
  -r, --n-rows N_ROWS   The number of rows to visualize (if applicable).
                        (default: 6)
  -c, --n-cols N_COLS   The number of columns to visualize (if applicable).
                        (default: 7)
  -t, --max-traversal MAX_TRAVERSAL
                        The maximum displacement induced by a latent
                        traversal. Symmetrical traversals are assumed. If
                        `m>=0.5` then uses absolute value traversal, if
                        `m<0.5` uses a percentage of the distribution
                        (quantile). E.g. for the prior the distribution is a
                        standard normal so `m=0.45` corresponds to an absolute
                        value of `1.645` because `2m=90%` of a standard normal
                        is between `-1.645` and `1.645`. Note in the case of
                        the posterior, the distribution is not standard normal
                        anymore. (default: 2)
  -i, --idcs IDCS [IDCS ...]
                        List of indices to of images to put at the begining of
                        the samples. (default: [])
  -u, --upsample-factor UPSAMPLE_FACTOR
                        The scale factor with which to upsample the image (if
                        applicable). (default: 1)
  --is-show-loss        Displays the loss on the figures (if applicable).
                        (default: False)
  --is-posterior        Traverses the posterior instead of the prior.
                        (default: False)
```

### Examples

Here are examples of plots you can generate:

* `python main_viz.py <model> reconstruct-traverse --is-show-loss --is-posterior` first row are originals, second are reconstructions, rest are traversals. Shown for `btcvae_dsprites`:

    ![btcvae_dsprites reconstruct-traverse](results/btcvae_dsprites/reconstruct_traverse.png)

* `python main_viz.py <model> gif-traversals` grid of gifs where rows are latent dimensions, columns are examples, each gif shows posterior traversals. Shown for `btcvae_celeba`:

    ![btcvae_celeba gif-traversals](results/btcvae_celeba/posterior_traversals.gif)

* Grid of gifs generated using code in `bin/plot_all.sh`. The columns of the grid correspond to the datasets (besides FashionMNIST), the rows correspond to the models (in order: Standard VAE, β-VAE<sub>H</sub>, β-VAE<sub>B</sub>, FactorVAE, β-TCVAE):

    ![grid_posteriors](results/grid_posteriors.gif)

For more examples, all of the plots for the predefined experiments are found in their respective directories (created using `./bin/plot_all.sh`).

## Data

Current datasets that can be used:
- [MNIST](http://yann.lecun.com/exdb/mnist/)
- [FashionMNIST](https://github.com/zalandoresearch/fashion-mnist)
- [3D Chairs](https://www.di.ens.fr/willow/research/seeing3Dchairs)
- [Celeba](http://mmlab.ie.cuhk.edu.hk/projects/CelebA.html)
- [2D Shapes / Dsprites](https://github.com/deepmind/dsprites-dataset/)

The dataset will be downloaded the first time you run it and will be stored in `data` for future uses. The download will take time and might not work anymore if the download links change. In this case either:

1. Open an issue
2. Change the URLs (`urls["train"]`) for the dataset you want in `utils/datasets.py` (please open a PR in this case :) )
3. Download by hand the data and save it with the same names (not recommended)

## Our Contributions

In addition to replicating the aforementioned papers, we also propose and investigate the following:

### Axis Alignment Metric

Qualitative inspections are unsuitable to compare models reliably due to their subjective and time consuming nature. Recent papers use quantitative measures of disentanglement based on the ground truth factors of variation **v** and the latent dimensions **z**. The [Mutual Information Gap (MIG)](https://arxiv.org/abs/1802.04942) metric is an appealing information theoretic metric which is appealing as it does not use any classifier. To get a MIG of 1 in the dSprites case where we have 10 latent dimensions and 5 generative factors, 5 of the latent dimensions should exactly encode the true factors of variations, and the rest should be independent of these 5.

Although a metric like MIG is what we would like to use in the long term, current models do not get good scores and it is hard to understand what they should improve. We thus propose an axis alignment metric AAM, which does not focus on how much information of **v** is encoded by **z**, but rather if each v<sub>k</sub> is only encoded in a single z<sub>j</sub>. For example in the dSprites dataset, it is possible to get an AAM of 1 if **z** encodes only 90% of the variance in the x position of the shapes as long as this 90% is only encoded by a single latent dimension z<sub>j</sub>. This is a useful metric to have a better understanding of what each model is good and bad at. Formally:

![Axis Alignment Metric](doc/imgs/aam.png)

Where the subscript *(d)* denotes the *d*<sup>th</sup> order statistic and *I*<sub>x</sub> is estimated using empirical distributions and stratified sampling (like with MIG):

![Mutual Information for AAM](doc/imgs/aam_helper.png)


### Single Model Comparison

The model is decoupled from all the losses and it should thus be very easy to modify the encoder / decoder without modifying the losses. We only used a single model in order to have more objective comparisons of the different losses. The model used is the one from [Understanding disentangling in β-VAE](https://arxiv.org/abs/1804.03599), which is summarized below:

![Model Architecture](doc/imgs/architecture.png)


## Losses Explanation

All the previous losses are special cases of the following loss:

![Loss Overview](doc/imgs/loss.png)

1. **Index-code mutual information**: the mutual information between the latent variables **z** and the data variable **x**. There is contention in the literature regarding the correct way to treat this term. From the [information bottleneck perspective](https://arxiv.org/abs/1706.01350) this should be penalized. [InfoGAN](https://arxiv.org/abs/1606.03657) get good results by increasing the mutual information (negative α). Finally, [Wasserstein Auto-Encoders](https://arxiv.org/abs/1711.01558) drops this term. 

2. **Total Correlation (TC)**: the KL divergence between the joint and the product of the marginals of the latent variable. *I.e.** a measure of dependence between the latent dimensions. Increasing β forces the model to find statistically independent factors of variation in the data distribution.

3. **Dimension-wise KL divergence**: the KL divergence between each dimension of the marginal posterior and the prior. This term ensures the learning of a compact space close to the prior which enables sampling of novel examples.

The losses differ in their estimates of each of these terms and the hyperparameters they use:

* [**Standard VAE Loss**](https://arxiv.org/abs/1312.6114): α=β=ɣ=1. Each term is computed exactly by a closed form solution (KL between the prior and the posterior). Tightest lower bound.
* [**β-VAE<sub>H</sub>**](https://openreview.net/pdf?id=Sy2fzU9gl): α=β=ɣ>1. Each term is computed exactly by a closed form solution. Simply adds a hyper-parameter (β in the paper) before the KL.
* [**β-VAE<sub>B</sub>**](https://arxiv.org/abs/1804.03599): α=β=ɣ>1. Same as **β-VAE<sub>H</sub>** but only penalizes the 3 terms once they deviate from a capacity C which increases during training.
* [**FactorVAE**](https://arxiv.org/abs/1802.05983): α=ɣ=1, β>1. Each term is computed exactly by a closed form solution. Simply adds a hyper-parameter (β in the paper) before the KL. Adds a weighted Total Correlation term to the standard VAE loss. The total correlation is estimated using a classifier and the density-ratio trick. Note that ɣ in their paper corresponds to β+1 in our framework.
* [**β-TCVAE**](https://arxiv.org/abs/1802.04942): α=ɣ=1 (although can be modified), β>1. Conceptually equivalent to FactorVAE, but each term is estimated separately using minibatch stratified sampling.

 ## Cite

When using one of the models implemented in this repo in academic work please cite the corresponding paper (linked at the top of the README). In case you want to cite this specific implementation then you can use:

```
@misc{dubois2019dvae,
  title        = {Disentangling VAE},
  author       = {Dubois, Yann and Kastanos, Alexandros and Lines, Dave and Melman, Bart},
  month        = {march},
  year         = {2019},
  howpublished = {\url{http://github.com/YannDubs/disentangling-vae/}}
}
```
