---
layout: post
title: Feeding text data in Tensorflow
tags: tf.data tensorflow TextLineDataset
---

In this post, we will look at how to **convert text data into a tensor**. [`tf.data`](https://www.tensorflow.org/programmers_guide/datasets) is the [recommended method](https://www.tensorflow.org/api_guides/python/threading_and_queues) to feed data to your tensorflow model and is [now core part](https://github.com/tensorflow/tensorflow/blob/master/RELEASE.md) of [tensorflow](https://www.tensorflow.org).

Some tensorflow developers might find this strange. Why do we need this `tf.data`? Isn't creating [placeholders](https://www.tensorflow.org/api_docs/python/tf/placeholder) and `feed_dict` the way to do it? Well, you can certainly use feed_dict, but that requires you to completely pre-process your data, write a separate batching function. Also,`feed_dict` [does not scale well](https://www.tensorflow.org/performance/performance_guide).
>While feeding data using a feed_dict offers a high level of flexibility, in most instances using feed_dict does not scale optimally. However, in instances where only a single GPU is being used the difference can be negligible. Using the Dataset API is still strongly recommended.

Surprisingly, I found [sparse and unclear documentation](https://www.tensorflow.org/versions/master/api_docs/python/tf/data/TextLineDataset) on how to use dataset API for text! I picked up how to use this powerful API from
[NMT blog](https://github.com/tensorflow/nmt) and [code](https://github.com/tensorflow/nmt/blob/master/nmt/utils/iterator_utils.py), and experimenting using the good old jupyter notebook. I have found this method of experimentation using notebook extremely useful for writing code in tensorflow. Thus, a more detailed version of this post is available as a notebook. Feel free to download the notebook, and add your own code to thoroughly understand how these APIs work.

To be absolutely concrete, this is what we will learn:
* You have a text corpus file with one sentence per line and a list of words you care about (generally referred to as vocabulary).
* We will learn how to create a data iterator that will return one sentence per call.
* We will extend this data iterator to return a list of tokens.
* We will further extend the data iterator to return a list of integer tokens.
* Finally, we will create a batch for this iterator, such that it returns token integers for a batch at a time!
* Text has an interesting property that each sentence can be of different length. We will also add a sentence length to our iterator.

OK. Let us get started!

### Download and explore data
Well, all text experiments begin with data. So let us setup data. We will work with a small file `sample.en` with 100 sentences in this post. Feel free to change `sample.en` to `train.en` and experiment.

* Download the english sentences `train.en` and vocabulary file `vocab.en` from [Ted Talks dataset IWSLT'15 English-Vietnamese data](https://nlp.stanford.edu/projects/nmt/). Further extract top 100 sentences to `sample.en`
  ```
  wget https://nlp.stanford.edu/projects/nmt/data/iwslt15.en-vi/train.en
  wget https://nlp.stanford.edu/projects/nmt/data/iwslt15.en-vi/vocab.en
  head -100 train.en > sample.en
  ```

* Check the number of lines and make sure you see same output as indicated below
  ```
  wc -l *.en

  100 sample.en
  133317 train.en
  17191 vocab.en
  150608 total
  ```
* Take a look at the first two lines of `sample.en`. You would notice that each line contains a single sentence. Further, each sentence seems tokenized.

  ```
  head -2 sample.en

  Rachel Pike : The science behind a climate headline
  In 4 minutes , atmospheric chemist Rachel Pike provides a glimpse of the massive scientific effort behind the bold headlines on climate change , with her team -- one of thousands who contributed -- taking a risky flight over the rainforest in pursuit of data on a key molecule .
  ```

* Now, take a look at first 10 lines of `vocab.en`. You would notice that each line contains one word.
  ```
  head vocab.en

  <unk>
  <s>
  </s>
  Rachel
  :
  The
  science
  behind
  a
  climate
  ```

### Code, for people in a hurry!
Well, yes I really love the [book](https://www.amazon.com/Astrophysics-People-Hurry-deGrasse-Tyson/dp/0393609391/ref=pd_cp_14_1?_encoding=UTF8&psc=1&refRID=HKQZWNV4ZBDWTKCCDPDR) and this title was *inspired* by it!

Also, some people find it useful to see the complete code with minimalist explanations before diving into details.
Here is the complete [notebook](https://github.com/vineetm/tensorflow-notes/blob/master/dataset/notebooks/tf-text-data-allcode.ipynb) for this code.
  ```python
  import tensorflow as tf
  import os
  from tensorflow.python.ops import lookup_ops

  #Replace `DATA_DIR` with folder where you downloaded the data. You can replace `sample.en` with `train.en`
  DATA_DIR = '.'
  sentences_file = os.path.join(DATA_DIR, 'sample.en')
  vocab_file = os.path.join(DATA_DIR, 'vocab.en')

  #lookup table, converts a token to integer. By default returns token at first line of `vocab.en`
  #Requires to be initialized using tf.tables_initializer inside a session.
  vocab_table = lookup_ops.index_table_from_file(vocab_file, default_value=0)

  #Creates a dataset which retruns a single sentence
  dataset = tf.data.TextLineDataset(sentences_file)

  #Converts each sentence to a list of tokens
  dataset = dataset.map(lambda sentence: tf.string_split([sentence]).values)

  #Converts list of tokens to list of token integers
  dataset = dataset.map(lambda words: vocab_table.lookup(words))

  #Adds length of sentence (number of tokens)
  dataset = dataset.map(lambda words: (words, tf.size(words)))

  #Convert to a batch of size 32. Padded batch appends 0 for shorter sentences.
  dataset = dataset.padded_batch(batch_size=32, padded_shapes=(tf.TensorShape([None]), tf.TensorShape([])))

  # Dataset iterator. Needs to be initialized
  iterator = dataset.make_initializable_iterator()
  ```

  Note, that all the above code does is creates some operators that generate a tensor. In this case tensor from a text file! To actually see the output, you would need to build the graph and execute it inside a session. Here is how you can do it ...
  ```python
  with tf.Session() as sess:
    sess.run(iterator.initializer)
    sess.run(tf.tables_initializer())

    sentences = sess.run(iterator.get_next())
    print ('#Num Sentences: %d Shape[txt]: %s Shape[len]: %s'%(len(sentences[0]), sentences[0].shape, sentence[1].shape))
    print ('S[%d]:%s Length:%d\n'%(1, sentences[0][1], sentences[1][1]))
    print ('S[%d]:%s Length:%d'%(14, sentences[0][14], sentences[1][14]))
  ```

  This would return:
  ```bash
  Num Sentences: 32 Shape[txt]: (32, 50) Shape[len]: (32,)
  S[1]:[11 12 13 14 15 16  3  0 17  8 18 19 20 21 22 23  7 20 24 25 26  9 27 14 28
   29 30 31 32 19 33 34 35 31 36  8 37 38 39 20 40 41 42 19 43 26  8 44 45 46] Length:50

  S[14]:[106  32  19  20 173  47 157 121 174   0  14 147 121 175  46 112 113   8
   176 177  45  46 178 179 180 181 182  19 143  46   0   0   0   0   0   0
     0   0   0   0   0   0   0   0   0   0   0   0   0   0] Length:30
  ```

Notice that shape of tokens part is **32x50** (`batch_size` x maximum tokens in the batch). Shape of length is **32** (one per element of batch). Also sentence 14 has length 30. The remaining portion is padded with 0.

Go ahead and try by calling iterator another time (Copy `sentences = sess...` line below!). This basically would return the second batch. What is the maximum length of sentence that you see? You might want to check the [notebook](https://github.com/vineetm/tensorflow-notes/blob/master/dataset/notebooks/tf-text-data-allcode.ipynb) for the correct answer.

### Deep Dive
If all the code above seems hurried (Well, it should!), here is a more slow paced explanation. As before, there is also a [notebook](https://github.com/vineetm/tensorflow-notes/blob/master/dataset/notebooks/tf-text-data.ipynb) where you get to run (and modify) the code to your heart's content!

#### Step 1: Data iterator for text
Lets us take a step back, and see concretely how iterator works

```python
#Creates a dataset which returns a single sentence
dataset = tf.data.TextLineDataset(sentences_file)
```
All this does is create a dataset that returns one line on each call. But how do we use it? We need an iterator...

```python
# Dataset iterator. Needs to be initialized
iterator = dataset.make_initializable_iterator()
```
Think of this as an operator which can **loop over your text file**. But as with all tensorflow operators, you cannot directly see its output. It does return a tensor though! Okay, so how can we see what it does?
```python
with tf.Session() as sess:
    sess.run(iterator.initializer)

    sentence = sess.run(iterator.get_next())
    print(sentence)
```

> b'Rachel Pike : The science behind a climate headline'

All we did inside a session was to initialize our iterator, and then call `get_next()`, which returns a sentence, the very first sentence in `sample.en`!


#### Step 2: Return a list of tokens

All we need to do to convert a string to a list of tokens is use [`tf.string_split()`](https://www.tensorflow.org/api_docs/python/tf/string_split). This operator returns many fields, we are interested in tokens so we use `values`. Also note we need to pass sentence as `[sentence]`.

```python
#We saw this before, just returns one line from text file
dataset = tf.data.TextLineDataset(sentences_file)

#This would convert a sentence to a list of tokens.
dataset = dataset.map(lambda sentence: tf.string_split([sentence]).values)

#Good, ol iterator!
iterator = dataset.make_initializable_iterator()
```

Okay, so what would be expect now? Note, there is no change in the code below!
```python
with tf.Session() as sess:
    sess.run(iterator.initializer)

    sentence = sess.run(iterator.get_next())
    print(sentence)
```
> [b'Rachel' b'Pike' b':' b'The' b'science' b'behind' b'a' b'climate'
 b'headline']

 That's right, a list of tokens!!  

#### Step 3: Lookup Table, and list of token *integers*
OK, so we have made some progress. We know how to create an iterator, and how to return a list of tokens.

To convert tokens (strings) to integers, we need a lookup table. Think of lookup table as a dictionary, which returns a unique index for each token (And a default index for out of vocabulary words)

```python
from tensorflow.python.ops import lookup_ops

# Returns a lookup op, which will convert a string to integer, as per vocab_file.
# By default this returns 0 (Look at what is first line of vocab_file)
vocab_table = lookup_ops.index_table_from_file(vocab_file, default_value=0)
```
So, the above code creates the dictionary (dictionary operator, not python dictionary!) such that we can convert a token (string) to integer.
```python
dataset = tf.data.TextLineDataset(sentences_file)
dataset = dataset.map(lambda sentence: tf.string_split([sentence]).values)

dataset = dataset.map(lambda words: vocab_table.lookup(words))
iterator = dataset.make_initializable_iterator()
```

And, what would be expect now?
```python
with tf.Session() as sess:
    sess.run(iterator.initializer)

    sentence = sess.run(iterator.get_next())
    print(sentence)
```
> [ 3  0  4  5  6  7  8  9 10]

We, finally have integers!!!!

#### Step 4: Append length of sentence (aka number of tokens)

Why would we need to append number of tokens. Well, text is interesting. Not all sentences are of same length. Thus if we are considering batching them together, we need to know where sentence ends. Useful for [`dynamic_rnn`](https://www.tensorflow.org/api_docs/python/tf/nn/dynamic_rnn).

```python
dataset = tf.data.TextLineDataset(sentences_file)
dataset = dataset.map(lambda sentence: tf.string_split([sentence]).values)
dataset = dataset.map(lambda words: vocab_table.lookup(words))

#We can compute length at runtime using tf.size
dataset = dataset.map(lambda words: (words, tf.size(words)))

iterator = dataset.make_initializable_iterator()
```

#### Final Step: Batching
This step involves grouping sentences together. This is where `tf.data` comes really handy, as it takes care of batching for you.

```python
dataset = tf.data.TextLineDataset(sentences_file)
dataset = dataset.map(lambda sentence: tf.string_split([sentence]).values)
dataset = dataset.map(lambda words: vocab_table.lookup(words))


dataset = dataset.map(lambda words: (words, tf.size(words)))

#First argument tells us what is size of batch. [None] says this is vectors of unknown size
dataset = dataset.padded_batch(32, padded_shapes=(tf.TensorShape([None]), tf.TensorShape([])))
iterator = dataset.make_initializable_iterator()
```

You execute it in same fashion. But get a batch of sentences now.
```python
with tf.Session() as sess:
    sess.run(iterator.initializer)
    sess.run(tf.tables_initializer())

    #This is now a tuple. Where [0] is sentence [1] is length
    sentences = sess.run(iterator.get_next())
  print(len(sentences[0]), len(sentences[1]))
```
