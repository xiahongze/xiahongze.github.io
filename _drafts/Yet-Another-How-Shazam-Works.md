---
layout: post
title: Yet Another Post About How Shazam Works
date:   2019-11-17 15:45:00 +1100
categories: tech
comments: true
---

This post is purely based on [Wang's paper](https://www.ee.columbia.edu/~dpwe/papers/Wang03-shazam.pdf).
I will try to make the explanation as straight-forward as possible. The entire process
consists of four major steps, as is followed.

1. Generate Constellation Map from Spectrogram for your library of audios
2. Generate fingerprints using the Constellation Map for each file
3. Repeat 1&2 for a sample audio of interest
4. Use the fingerprints of the sample audio to search the library

## Concepts

### Spectrogram

Spectrogram is nothing but frequency distribution in the time domain. Given a
time series of signal, we could use Fourier transformation to transform a moving
window of the signal and eventually map
$$
\operatorname{Signal}(t)\rightarrow\operatorname{Freq}(t)
$$
.

### Constellation Map

Constellation Map is an invented concept from Wang's paper, quoted

> We term the sparse coordinate lists “constellation maps” since the coordinate
> scatter plots often resemble a star field.

It is actually just a reduced spectrogram --- only the top frequency
components at each time step is preserved.

### Footprints

Footprints are signatures of a piece of audio, generated from the combinatorial
hashing of frequencies and time deltas, track ID and start times.
I will explain this in another section.

## Constellation Map Generation

First, we need to generate the spectrogram for a piece of audio.

Input:

- signal, $$\operatorname{Signal}(t)$$

Parameters:

- sample rate, like, 44.1KHz
- window function

Output:

- Spectrogram, $$\operatorname{Freq}(t)$$
- shape: (time, freq)
- implicitly, a frequency array is generated, $$A_{freq}$$


For more information, one can read the [doc from scipy](https://docs.scipy.org/doc/scipy/reference/generated/scipy.signal.spectrogram.html).


Reducing the Spectrogram along the frequency axis is simple.

$$
\operatorname{ConstellationMap}(t) =
A_{freq}[
\operatorname{Argmax}(\operatorname{Freq}(t), axis=0)[\mathrm{:,:N}]
]
$$

Input:

- top N components
- Spectrogram

Output:

- Constellation Map, $$\operatorname{ConstellationMap}(t)$$
- shape: (time, N)

## Footprint Generation

### Combinatorial Hashing

According to Wang's paper, a pair of frequency-time is used to generate a hash.
Relative time is used instead of absolute timestamps given by the frequency-time
pair. Hence, three numbers, namely $$f_0$$, $$f_1$$ and
$$\delta t=t_1 - t_0$$
are combinatorially hashed to generate a hash.

If $$\mathcal{H}$$ is the hash function and $$h_i$$ and $$n_i$$ is the hash and
the number at step i, the combinatorial hashing can be written as

$$h_i = \mathcal{H}(h_{i-1}, n_i)$$

Or programmatically, let's define a combinatorial hashing function $$\mathrm{H}$$ as

```python
def H(hfunc: Callable, numbers: List[int]) -> int:
    r = 0
    for n in numbers:
        r = hfunc(r, n)
    return r
```

The reason for using frequency-time pairs instead of one single frequency-time
point is that this greatly improves the specificity. Quotes from the paper,

> We see that by using combinatorial hashing, we have traded off approximately 
> 10 times the storage space for approximately 10000 times improvement in speed,
>  and a small loss in probability of signal detection.

### Choosing Frequency-Time Pairs

Given an anchor point in the Constellation Map, one can define his target zone
after that anchor point. Hence, the anchor point and the points in the target
zone forms frequency-time pairs.

```python
def selectAnchors(freqs: List[int]) -> List[int]:
    """select frequencies of interest. This could be an identity function"""
    pass

def findTargetPoints(freq: int, i_t: int, constellationMap: List[List[int]],
    times: List[int]) -> List[Tuple[int, int]]:
    """given an anchor point (freq, idx at time) and the constellation map,
    returns points in the target zone"""
    pass

def makeFrequencyTime(constellationMap: List[List[int]], times: List[int]) \
    -> Tuple[Tuple[int, int], Tuple[int, int]]:
    """returns (f0,t0), (ft, tt)"""
    for i, (t, freqs) in enumerate(zip(constellationMap, times)):
        for anchor in selectAnchors(freqs):
            for point in findTargetPoints(anchor, i, constellationMap. times):
                yield (anchor, t), point
```

### Components of a Fingerprint

$$
\mathrm{fingerprint} \equiv [\mathrm{hash:track id:starttime}]
$$

In Wang's paper, a fingerprint is represented by a 64 bits integer, where the
first 32 bits is the hash and the second half is the track id plus the start
time of the anchor point. However, one can choose what makes sense to his own
application.

## Finding a Match

As is discussed, a song can be represented by a number of fingerprints generated
from the Constellation Map. Finding a match in the library given a sample audio
becomes a process of hash lookup.

```python
def lookup(sample, library_hashes):
    matches = []
    for fingerprint in makeFingerprints(sample):
        # match could be just the second half of the 64 bit construct
        match = library_hashes.search(fingerprint)
        if match:
            matches.append((fingerprint, match))
    return matches
```

From all the matches returned by the library, one could group them by the track
IDs. One simple way of scoring the results can be the match counts for each
track. However, it was proven to be not as robust as one expects. I did read
other people claim that it was actually adequate. At least, I would say the
performance could vary from case to case.

What was proposed in Wang's paper is that one should rely on the scatter plot of
time pairs constructed from the sample audio and the song from the library. All
information should be contained in the `matches` returned by the function
`lookup`. Quotes from the paper,

> If the files match, matching features should occur at similar relative offsets
> from the beginning of the file, i.e. a sequence of hashes in one file should
> also occur in the matching file with the same relative time sequence. The
> problem of deciding whether a match has been found reduces to detecting a
> significant cluster of points forming a diagonal line within the scatterplot.
> Various techniques could be used to perform the detection, for example a Hough
> transform or other robust regression technique.

However, as stated in the paper, these techiques are usually time consuming and
an alternative approach was proposed as an approximated solution to the problem.

The suggested approach, which can be easily told from the scatter plot, is that
if one plots the histogram of $$\delta t=t_1 - t_0$$ from the matches, one would
expect a sharp peak somewhere for a true match, whereas a false match would be
flat. In this way, the problem of scoring matches is reduced to 
$$\mathcal{O}(n\operatorname{log}n)$$
.