---
title: Pitch Detection
date: 2020-06-06T16:52:17.000+05:30
author: Abhishek Nandekar
Description: ''
Tags: []
Categories: []
menu:
  overview:
    parent: main
    weight: 100
draft: true

---
## An algorithm for detecting the pitch of a given audio track

### Introduction

Pitch is a fundamental feature for the production of music, and extracting instantaneous pitch of an audio signal can be helpful for defining the melody of the signal, which can further help in identifying melodic patterns in a song. Such melodic patterns have been used for devising a feature set\[^MiningPatterns\] useful for mathematically modelling rāgas in Indian Classical Music as vector spaces\[^ragaVector\].

Pitch is analogous to _primary colours_ (red, yellow and blue) in painting. It is the building block of a song's melody. A pitch can be separated into two components, which are referred to as **tone height** and **chroma**. The tone height refers to the octave number and the chroma to the respective pitch spelling attribute contained in the set {C, C♯, D, $\\dots$, B}.

### Techniques Used

#### Short Time Fourier Transform(STFT)

As explained in \[^mller_2016\], let $x:\\mathbb Z\\rightarrow \\mathbb R$ be a real valued discrete-time signal obtained by equidistant sampling with respect to a fixed sampling rate $f_s$. Let $w:\[0: N-1\] \\rightarrow \\mathbb R$ be the sampled window function, such that $N \\in \\mathbb N$. In case of a rectangular window,

$$\\begin{align}
w(n) :=
\\begin{cases}
1, \~\~\~\\forall n \\in \[0, N-1\] \\\\
0, \\text{ otherwise}
\\end{cases}
\\end{align}$$

Here, $N$ is also useful for determining the window length($N/f_s$). There exists another parameter, $H \\in \\mathbb N$, known as the **hop size**. This is the step size, in which the window is shifted across the signal.

#### Constant Q-Transform(CQT)

Constant Q-Transform(CQT) is a technique that transforms the discrete-time-signal $x(n)$ into the time-frequency domain such that the center frequencies
of the frequency bins are geometrically spaced and their Q-factors are all equal. It is essentially a wavelet transform with high Q-factors, such as 12-96 bins per octave.

In LibROSA, the Constant Q-Transform is calculated as follows. For a more detailed explanation, refer \[^Schrkhuber2010CONSTANTQTT\].

Here, every frequency bin has a different window length $N_k$. $w(n)$ is the window function, and $H$ is the hop-size.  
$$
\\begin{equation}
\\tag{CQT}
\\chi_{CQ}(k, n) := \\sum_{j = n - \\lfloor N_k/2 \\rfloor}^{n + \\lfloor N_k/2 \\rfloor} x(j)\\cdot a_k^{*}(j - n + N_k/2)
\\end{equation}$$

where, $k \\in \[1:K\]$ is the frequency bin index,
and $a_k^{*}$ is the complex conjugate of $a_k$, also known as the
"time-frequency atom", defined as, $$\\begin{aligned}
a_k(n) := \\frac{1}{N_k}w(\\frac{n}{N_k})\\exp{
\\bigg(-i2\\pi n \\frac{f_k}{f_s}\\bigg)}\\end{aligned}$$ where, $f_k$ is
the center frequency of the $k^{th}$ bin, $f_s$ is the sampling rate.
The window lengths $N_k \\in \\mathbb R$ are the window lengths, and are
inversely proportional to $f_k$, in order to have the same Q-factor for
all the bins.

Here, $$\\begin{aligned}
f_k := f_1\\cdot 2^{\\frac{k-1}{B}}  \\end{aligned}$$ where, $f_1$ is
the centre frequency for the lowest frequency bin, $B$ is the number of
bins per octave.

Then, the Q-factor for each bin($Q_k$) is defined as, $$\\begin{aligned}
Q_k := \\frac{f_k}{\\Delta f_k} = \\frac{N_k f_k}{\\Delta\\omega f_s}\\end{aligned}$$
where, $\\Delta f_k$ is -3 dB bandwidth frequency response of the atom
$a_k(n)$, and, $\\Delta\\omega$ is -3 dB bandwidth of the mainlobe of the
spectrum of the window function $w(n)$.

Since $Q_k$'s are constant for all the frequency bins, we can just call
it $Q$.

Selecting a low value for $Q$ leads to frequency smearing, since
$\\Delta f_k$ for each bin is high. But on the other hand, selecting a
very high value for $Q$ has a consequence that the portion of the
spectrum between the bins will not be analysed.

Hence, optimal value of $Q$ is calculated as, $$\\begin{aligned}
Q = \\frac{q}{\\Delta\\omega (2^{1/B} - 1)}\\end{aligned}$$ where $q$ is
known as the scaling factor, $0 < q \\leq 1$. Usually, $q$ is chosen to
be 1. $q < 1$ can be used to improve the time-resolution at the cost of
degrading the frequency resolution. The above equation can be
substituted in the definition of $Q$ and can be solved for $N_k$ to
obtain, $$\\begin{aligned}
N_k = \\frac{qf_s}{f_k(2^{1/B} - 1)}\\end{aligned}$$ To enable signal
reconstruction from the CQT coefficients, successive atoms can be placed
$H_k$ samples apart(Hop Size). Typically, $0<H_k\\leq\\frac{1}{2}N_k$

### Methodology

This experiment is essentially a reproduction of one of the methods discussed in \[^dl4mir\]. The training data set is a synthesised one.

In the original research, the training set is a pure sinusoid on 12 different pitches, starting from 440Hz. A **Constant Q-Transform (CQT)** is computed for each signal in the set. The first frame is chosen, and its power-output is converted into **the deciBel(dB) scale**. The vector is then normalised and fed to the network.

In our experiment, a set of pure sine waves over a wider range of pitches (4 octaves) is used as the training data. A magnitude-CQT is computed in one iteration of the experiment, while, a magnitude-STFT is computed in the other. We then choose the first frame of the output vector, normalise the values and feed it to the network. We skip the _power conversion to deciBel_ step, since it's inclusion was **heavily reducing the training accuracy.**

#### Training Model

The training model used in this experiment is fairly simple. There is only a single dense hidden layer, with no bias, which connects the $n$ dimensional input to 48 dimensional output. Here,

$$
\\begin{aligned}
n = \\begin{cases}
84, \~\~\\text{CQT} \\\\\\
1025, \~\~ \\text{FFT}
\\end{cases}
\\end{aligned}  
$$

The model uses the **Stochastic Gradient Descent(SGD)** optimiser, with **softmax** activation  and **categorical crossentropy** loss\[^dlbook\].

The validation set is generated by the same class that generated the data set. Since the data is synthetically generated, cross-validation was not necessary.

### Results

After the training process, the results obtained are

\[^MiningPatterns\]: S.  Gulati,  J.  Serr\`a,  V.  Ishwar,  and  X.  Serra.   Mining  melodic  patterns  inlarge  audio  collections  of  indian  art  music.   In2014 Tenth InternationalConference on Signal-Image Technology and Internet-Based Systems, pages264–271, 2014.

\[^ragaVector\]: S.  Gulati,  J.  Serr\`a,  V.  Ishwar,  S. Sent ̈urk,  and  X.  Serra.   Phrase-based rĀga recognition using vector space modeling.  In2016 IEEE InternationalConference on Acoustics, Speech and Signal Processing (ICASSP),  pages66–70, 2016

\[^dl4mir\]: Keunwoo  Choi,  Gy ̈orgy  Fazekas,  Kyunghyun  Cho,  and  Mark  B.  San-dler.   A  tutorial  on  deep  learning  for  music  information  retrieval.CoRR,abs/1709.04396, 2017

\[^mller_2016\]: Meinard M ̈uller.Fundamentals of Music Processing – Audio, Analysis, Al-gorithms, Applications.  Springer Verlag, 2015
\[^Schrkhuber2010CONSTANTQTT\]: Christian Sch ̈orkhuber.  Constant-q transform toolbox for music process-ing.  2010

\[^dlbook\]: Ian Goodfellow, Yoshua Bengio, and Aaron Courville. 2016. Deep Learning. The MIT Press.