![image](https://github.com/joy-void-joy/alexplainable/assets/56257405/7db54cd9-ab71-4898-8fb6-83732476fc8e)

# Alexplainable: Constructing an ImageNet recognizer to be explainable

[sorted segmented training dataset and eval dataset to be released]

Alexplainable is an attempt at making a semi-transparent [ImageNet](https://www.image-net.org/) recognizer using [AI-Pull-Requests](lesswrong post). Runs of this project will be had at [https://github.com/joy-void-joy/alexplainable-runs](https://github.com/joy-void-joy/alexplainable-runs).

Bet on [Manifold](https://manifold.markets/news/will-constructability-actually-work) about how much this process will succeed, and like us on [Manifund](manifund) to support our project.

General structure
===
The general purpose of this project is to investigate [constructability](lesswrong post): How feasible is it right now to construct AI projects from scratch in a way that makes it easy to understand what the different parts are.
We want to see how many classes of ImageNet we are able to construct a recognizer for, and how much human intervention does it require.

Our approach is the following:
- Define a graph of features using LLMs on images iteratively. Each node would be a feature that is directly observable and categorizable (like “bucket” or “rusty”)
- Segment the images, and classify the segments according to what nodes they activate.
- Train shallow convolutional neural networks on leaf nodes. When they have a good enough accuracy, have them commit a pull-request for integration
- Train convolutional neural network of ~1 layers for a feature at level (n+1) using only as input the feature-map of level n.
- If a node cannot be trained to have good enough accuracy in a fixed amount of time, use LLMs to refactor the graph further.

Early results
===

Defining an ontology
---
Our process for defining an ontology has been to:
- Start from a pre-defined ontology made in 30 minute by hand
- Have claude describe an image thoroughly
- Extract a graph from the description
- Have it merge the current graph with the extracted graph.
 
Doing this, we have been able to extract from 10 images [an ontology that seems to make sense](https://github.com/joy-void-joy/alexplainable/blob/main/early_results/ontology):

![image](https://github.com/joy-void-joy/alexplainable/assets/56257405/e3dfda63-f3ed-4abd-adbd-d9710ab38e12)

Some extract of them seems pretty solid and exactly like what we want:
```
│   ├── temporary fence
│   │   └── signs posted
│   │       ├── construction company information
│   │       └── safety warnings
```
Which is exactly what we want: a set of reviewable features that clearly compose together in a clear fashion.

This loop has two main problems however: some node needs to be cleaned up, like  “dense” or “timestamp”, and it currently grows linearly in the number of images:

![image](https://github.com/joy-void-joy/alexplainable/assets/56257405/38187ac5-259b-4933-9c14-0977db6a5a54)

We are still testing way to have Claude or other system refactor the ontology. For instance, it may be possible that using [function calls](https://docs.anthropic.com/claude/docs/tool-use) with Claude will yield better results than it did with GPT-4, and that this would make the ontology more succint.

Segmenting
---
Segmenting seems to work very well with [segment anything](https://github.com/facebookresearch/segment-anything):

![image](https://github.com/joy-void-joy/alexplainable/assets/56257405/7c652491-6cc5-4e09-a70b-614d66f3c8d7)

Classification of the segments with LLaVA would be cheap, but has been mixed. Claude has been able to classify segments fairly well, but also makes mistake:

![image](https://github.com/joy-void-joy/alexplainable/assets/56257405/070ad9c2-9bff-4808-b44b-b77f8557bdb4)

We have resorted to sorting the segments by hand for this prototype.

Training networks
---
We have trained networks to recognize Imagenet class n11939491. We have used the following ontology:

- n119
  - Flower head
    - disk
    - petal
  - Stem
  - Leaf

A network is composed of a CNN stack part (three layers of CNN), and of a small (4x1) linear head that decodes the CNN output to a probability. During training, a network learns to recognize its class against others (for instance, the petal network learns to discriminate between petal and non-petals like disks, flower head, n119, stem and leaf).
We then use the CNN output of a network to feed into the upper ones, the flower head network is only trained on the output of the disk and petal network. 

Using this we have been able to attain 75% accuracy on n119 pretty reliably.

We have tried using AvgPool or MaxPool instead of a linear network, however, we found their learning rate was way too eratic and did not converge quickly. After a quick look at the learned networks, it may be the case that linear networks are needed for networks to converge, and that we can swap to MaxPool after that.

Interpreting them
---
Unfortunately, it seems like some networks have not learned the proper task when reused globally:

![image](https://github.com/joy-void-joy/alexplainable/assets/56257405/59123ad9-d066-482c-8144-21ce8d8dafed)
![image](https://github.com/joy-void-joy/alexplainable/assets/56257405/1f9fc684-2173-411c-bf27-677783dcd23c)
Visualization of a disk network on the above input image

While others do show the correct behaviour:

![image](https://github.com/joy-void-joy/alexplainable/assets/56257405/c2fcbc2d-d989-4cac-bc55-371866abed1b)
Visualization of a petal network on the same image

We believe it should be possible to use the PR feedback loop we outline to filter for the networks that have learned the right thing, and then use automatically written plain code to compose them so that we have better understanding of what is happening.

Feature visualization
---
We wanted to have feature visualization of the trained networks so that they get automatically pull-requested in a fashion that would look like this:

![image](https://github.com/joy-void-joy/alexplainable/assets/56257405/1cdc0338-db8d-488a-b866-68204fb927d0)
Example taken from salient-imagenet

In an automated system, LLMs would oversee those PR and integrate them into an automatic branch. Then, when certain benchmarks are passed (like gaining 5% accuracy on a class, for instance), another PR gets posted to merge with master, which humans (or other AIs) have to review.

For now, the resulting feature visualization of the networks we have trained are not interpretable:

![image](https://github.com/joy-void-joy/alexplainable/assets/56257405/9a8805fc-09bd-47c1-94cc-5c4c5858da1d)
Feature visualization of petal, leaf and flower center disk network. The last one is almost interpretable, showing that it looks for yellow disks

As the petal and disk dataset vary greatly in colors and features, we would need to decompose the ontology further, as outlined above, to refine the different kinds of flowers that are in n119.
