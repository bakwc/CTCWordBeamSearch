# CTC Word Beam Search Decoding Algorithm

**CTC decoder with dictionary and language model for TensorFlow | C++ implementation | Python implementation**

## A First Example

The following code-skeleton gives a first impression of how to use the decoding algorithm with TensorFlow.
More details can be found in the Usage section. 

```python
# load compiled custom op
module=tf.load_op_library('TFWordBeamSearch.so') 

# decode mat using automatically created dictionary and language model
# corpus, chars, wordChars are (UTF8 encoded) strings and mat is a tensor (TxBxC)
beamWidth=25
lmType='NGrams'
lmSmoothing=0.01
decode=module.word_beam_search(mat, beamWidth, lmType, lmSmoothing, corpus, chars, wordChars) 

# feed matrix (TxBxC), evaluate and output decoded text (BxT)
res=sess.run(decode, { mat:feedMat }) 
```


## Introduction

Word beam search decoding is a Connectionist Temporal Classification (CTC) decoding algorithm.
It is used for sequence recognition problems like Handwritten Text Recognition (HTR) or Automatic Speech Recognition (ASR).
The following illustration shows a HTR system with its Convolutional Neural Network (CNN) layers, Recurrent Neural Network (RNN) layers and the final CTC (loss and decode) layer.
Word beam search decoding is placed right after the RNN layers to decode the output, see the red dashed rectangle in the illustration.

![context](./doc/context.png)

## Algorithm

The recognized words are constrained by a dictionary.
However, there is a free-mode such that strings of non-word characters like numbers and punctuation marks are also recognized.
A word-level Language Model (LM) can optionally be enabled. 
This algorithm is well suited when a large amount of words to be recognized is known in advance.
An overview of the algorithm is given in the illustration below.

![overview](./doc/overview.png)


The RNN output is fed into the algorithm and decoded. 
Textual input enables word beam search decoding to create a dictionary and LM.
Different settings control how the LM scores the beams (text candidates) and how many beams are kept per time-step.
The algorithm outputs the decoded text.

## Contents

This repository contains:

* Data: some samples from the IAM and Bentham HTR datasets are included
    * chars.txt: all characters (not including the CTC-blank) which are recognized by the RNN
    * wordChars.txt: characters which occur in words
    * corpus.txt: the text from which the dictionary and LM is created
    * mat_X.csv and gt_X.txt: RNN output generated by HTR system and ground truth
* C++ implementation
    * TensorFlow custom op: library which can be integrated in a TensorFlow computation graph
    * TensorFlow test program: Python script which uses the custom op and the test data to verify that everything works
    * Test program: executable which reads test data and decodes it. Not needed for TensorFlow, however, developing and debugging is much easier this way
* Python implementation: the prototype implementation of the algorithm. This is the best place to try new things or play around with the algorithm without having to think about the C++ compiler or TensorFlow
* Paper: gives a detailed explaination of the algorithm and evaluates it using the Bentham HTR dataset

## Usage

This section explains how to compile the TensorFlow custom operation, how to test it and how to use it. 
The C++ code can also be compiled into an executable.
Further, a prototype of the algorithm is implemented in Python without any dependency to TensorFlow.

### TenorFlow Custom Op

#### 1. Compile

Go to the ```cpp/proj/``` directory and run the script ```buildTF.sh```.
This creates a library object (Linux only, tested with Ubuntu 16.04 and TensorFlow 1.3.0).

#### 2. Test Custom Op

Then go to the ```tf/``` directory and run the script ```python testCustomOp.py```.
The expected output is as follows:

```text
Mini example:
Label string:  [1 0 3]
Char string: "ba"

Real example:
Label string:  [76 78 59 70 66 77 77  0 59 72 77 65  0 70 62 71 77 58 69  0 58 71 61  0 60
 72 75 73 72 75 62 58 69 10  0 66 76  0 63 58 75  0 59 62 82 72 71 61  0 58
 71 82  0 66 61 62 58 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93
 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93 93]
Char string: "submitt both mental and corporeal, is far beyond any idea"
```

#### 3. Add to your TensorFlow Model

The script ```tf/testCustomOp.py``` is fully documented.
A high-level overview of the inputs and output was already given.
Here follows a more technical discussion.
The interface of the operation is: ```word_beam_search(mat, beamWidth, lmType, lmSmoothing, corpus, chars, wordChars)```.
Some notes regarding the input parameters:

* Input matrix (mat): is expected to have shape TxBx(C+1) with the **softmax-function already applied** (in contrast to the TensorFlow operations ctc_greedy_decoder and ctc_beam_search_decoder!). The CTC-blank must be the last entry in the matrix.
* Beam Width (beamWidth): number of beams which are kept per time-step
* Scoring mode (lmType): pass one of the four strings (not case-sensitive). The running time with respect to the dictionary size W is given.
    * "Words": only use dictionary, no scoring: O(1)
    * "NGrams": use dictionary and score beams with LM: O(log(W))
    * "NGramsForecast": forecast (possible) next words and apply LM to these words: O(W*log(W))
    * "NGramsForecastAndSample": restrict number of (possible) next words to at most 20 words: O(W)
* Smoothing (lmSmoothing): LM uses add-k smoothing to allow word pairs which are not known from the training text, i.e. for which the bigram probability is zero. Set to values between 0 and 1, e.g. 0.01. To disable smoothing, set to 0
* Text (corpus): is given as a UTF8 encoded string. The operation creates its dictionary and (optionally) LM from it
* Characters (chars): must be given as a UTF8 encoded string. If the number of characters is C, then the RNN output must have the size TxBx(C+1) with the last entry representing the CTC-blank label. The ordering of the characters must correspond to the ordering in the RNN output, e.g. if the RNN outputs the probabilities for "a", "b", " " and CTC-blank in this order, then the string "ab " must be passed
* Word characters (wordChars): define how the algorithm extracts words from the text. Must be passed as a UTF8 encoded string. If the word characters are "ab", and the text "aa ab bbb a" is passed, then the words "aa", "ab" and "bbb" will be extracted and used for the dictionary and the LM. Of course, the word characters must be a subset of the characters recognized by the RNN, i.e. 0<len(wordChars)<len(chars)


This code snippet shows how to load the custom op and how to use it.

```python
# the RNN output has one additional character (the CTC-blank)
assert(len(chars)+1==mat.shape[2])

# load custom TF op
word_beam_search_module=tf.load_op_library('../cpp/proj/TFWordBeamSearch.so')

# decode node using the "Words" mode of word beam search
decode=word_beam_search_module.word_beam_search(mat, 25, 'Words', 0.0, corpus.encode('utf8'), chars.encode('utf8'), wordChars.encode('utf8'))

# feed matrix of shape TxBxC and evaluate TF graph
res=sess.run(decode, { mat:feedMat })
```

The output of the algorithm has shape BxT. 
The **label strings** are **terminated by a CTC-blank** if the length is smaller than T, similar as a C string (in contrast to the TensorFlow operations ctc_greedy_decoder and ctc_beam_search_decoder which use a SparseTensor!).
The following illustration shows an output with B=3 and T=5. 
"-" represents the CTC-blank label.

![output](./doc/output.png)

The following code snippet shows how to get the label string of the first batch element and how to transform it into a character string.

```python
blank=len(chars)
s=''
batch=0
for label in res[batch]:
	if label==blank:
		break
	s+=chars[label]
```

### C++ Test Program

Go to the ```cpp/proj/``` directory and run the script ```build.sh``` or open ```WordBeamSearch.sln``` with Visual Studio.
This creates an executable (Linux and Windows).
The expected output is as follows:

```text
Sample: 1
Result:       "brain."
Ground Truth: "brain."
Accumulated CER and WER so far: CER: 0 WER: 0
Average Time: 93ms

Sample: 2
Result:       "supposed"
Ground Truth: "supposed"
Accumulated CER and WER so far: CER: 0 WER: 0
Average Time: 70ms

Sample: 3
Result:       "submitt both mental and corporeal, is far beyond any fa"
Ground Truth: "submitt, both mental and corporeal, is far beyond any idea"
Accumulated CER and WER so far: CER: 0.0555556 WER: 0.0833333
Average Time: 67ms

Press any key to continue
```

### Python 

Go to the ```py/``` directory and run the script ```python main.py``` (tested on Windows and Linux, Python 2 and 3).
The expected output is as follows:

```text
Decoding 3 samples now.

Sample: 1
Filenames: ../data/bentham/mat_0.csv|../data/bentham/gt_0.txt
Result:       "brain."
Ground Truth: "brain."
Editdistance: 0
Accumulated CER and WER so far: CER: 0.0 WER: 0.0

Sample: 2
Filenames: ../data/bentham/mat_1.csv|../data/bentham/gt_1.txt
Result:       "supposed"
Ground Truth: "supposed"
Editdistance: 0
Accumulated CER and WER so far: CER: 0.0 WER: 0.0

Sample: 3
Filenames: ../data/bentham/mat_2.csv|../data/bentham/gt_2.txt
Result:       "submitt both mental and corporeal, is far beyond any fa"
Ground Truth: "submitt, both mental and corporeal, is far beyond any idea"
Editdistance: 4
Accumulated CER and WER so far: CER: 0.0555555555556 WER: 0.0833333333333
```

## Algorithm Details

Interested in how the algorithm works?
Have a look into [this paper](doc/report.pdf).