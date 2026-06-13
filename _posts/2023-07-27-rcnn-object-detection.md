---
title: "What is Object Detection and how do R-CNN models do it?"
last_modified_at: 2023-07-27
categories:
  - machine-learning
  - high-school
author: Jaden Mu
mathjax: true
---

### What is Object Detection
You probably know what image classification is - a model takes an image with some kind of object and labels what the object is.  Object Detection is slightly more complex.  Instead of an image with only one object, object detectors have to deal with images with multiple objects, separately labeling what each object is.  Object detectors also have to localize each object, drawing a bounding box around each object.

![Classification vs Detection](/assets/Mini-RCNN/classification_vs_detection.jpeg)

### Intro + Why RCNN?
Recently I've been working on finetuning YOLO models for a paper I'm hoping to submit for ISEF.  The Ultralytics package provides a lot of streamlined high-level APIs, so finetuning and inference can be done without knowing anything about the model architecture, but I was nonetheless curious about what was actually going on under the hood.  YOLO, is of course, just one model architecture that can be applied to Object Detection - RCNN, Fast/Faster RCNN, and SSD models fill similar roles.

YOLO stands for You Only Look Once, and it offers excellent performance at real-time speeds.  "You Only Look Once" means that an image only passes through one neural network, so it's much faster than two-stage detectors like RCNN and Faster RCNN.

I thought that actually implementing an object detector (nearly) from scratch would be an interesting side project, but after spending quite a bit of time reading and rereading the original YOLO paper, I decided that the YOLO model was just too difficult for me to implement from scratch.  The YOLO model is actually quite hard to understand conceptually, because it needs to perform the same task as a two-stage detector with only one step.  Instead, I'm choosing to implement RCNN, which was first proposed 2013.  Fast/Faster-RCNN are both later improvements on the original RCNN architecture, but they all use a 2-stage process and are therefore slower than YOLO and SSD.  

However, I actually think that RCNN is a more interesting model to implement from scratch - they use some really cool algorithms in conjunction with ML to solve a difficult problem.

### Sliding Window Models - Leading up to the RCNN
While image classification and object detection are two different tasks, they do share clear similarities.  Your first thought might therefore be to apply image classification models to object detection.  Since classification models can't draw bounding boxes or label multiple objects, we'll need to use an algorithm called sliding window search.  I like to think of sliding window search as being similar to a convolution operation, but with a classifier model taking in an image of dimensions N x M replacing the N x N kernel.  We run the image classifier model on each N x M "window" of the image, using the N x M window as the bounding box if an object is detected.  This window slides across the image by some step size so that the entire image can be processed.

![Sliding Window](/assets/Mini-RCNN/sliding_window_example.gif)

In the example above, a classifier for eyes, noses, and lips can be run on each window, which eventually "slides" over those objects.  Every time the classifier labels an eye, nose, or lip in a window, we save the position of the window and the label from the classifier.

Sliding window models are indeed somewhat capable and were the SOTA in the early 2000s, like the Viola-Jones framework for detecting faces.

### Problems with Slinding Window Detectors
However, sliding window detectors suffered from several problems - chief among them the inflexible window sizes.  In the real world, objects have diverse aspect ratios - a human and a car don't have the same aspect ratio, and therefore can't both be effectively captured by the same window size.  Additionally, large window sizes can introduce the possibility of multiple objects we want to detect being in the same window, confusing our classifer which was designed to only label individual objects.  The solution is then to slide multiple windows across the image - we would need to slide windows of different sizes, along with windows of different aspect ratios for each size (these sizes and aspect ratios are often determined by a human for *every* single object class).  We would then need some sort of way to remove duplicate detections from our many, many sliding windows which will overlap with each other.  That means that hundreds of thousands or often millions of windows must be classified - an exhaustive search is performed for every possible position of every window on the image.  That's why the peak of sliding window detectors was in the early 2000s, when fast algorithms like SVMs and random forest classifiers along with hand-coded image features were still the primary methods used for image classification - running a more performant but slower CNN would be completely infeasible on so many samples.  Of course, sliding window detectors are still a reasonable solution if there are very few object classes - fewer object classes means fewer aspect ratios for windows.

### The RCNN
The RCNN, standing for Region-based Convolutional Neural Network, sought to address the speed and aspect ratio problems of sliding window detectors, while also harnessing the power of CNNs over simpler classifiers.

Instead of sliding windows of different sizes over every possible region, RCNN uses a region proposal algorithm to generate windows of the full image that are likely to contain objects.  A CNN is then used to extract features from each region, which are fed into a SVM image classifier.  You can think of the region proposal stage as generating possible bounding boxes (regions of interest) and the classification stage as filtering and labeling the possible bounding boxes.  RCNNs offered a massive performance improvement over sliding window detectors because their region proposal stage allowed slower CNNs to be applied to the relatively fewer windows (around 1000 compared to potential millions for sliding window detection).

Of course, RCNNs have their own issues.  Most region proposal algorithms are hand-designed and aren't trained on a dataset in conjunction with the CNN and the SVM, so they can generate poor proposals (and be hard to adapt to a specific setting).  The most popular region proposal algorithms also generate a lot of proposals - although 2000 is a lot better than the number of windows from sliding window detectors, RCNNs can't even come close to real-time detection. Additionally, the final SVM classifier, although fast, is not ideal - we'd much prefer to just use a neural network end-to-end for classification.  The entire model is also obviously not differentiable.  These issues are all solved in clever ways in Fast-RCNN and Faster-RCNN, but I really like the conceptual simplicity of RCNNs and their use of 3 different algorithms.
![RCNN Architecture Diagram](/assets/Mini-RCNN/rcnn_architecure.png)

### RCNN Region Proposal: Edge Boxes and Selective Search
All region proposal algorithms need to return a list of windows that are likely to contain objects.  Of course, region proposal algorithms are not perfect, so they have to balance between precision and recall like any other model (where a true positive is if a proposed window actually contains an object).  Precision is equal to the algorithm's $$\frac{\text{True Positives}}{\text{True Positives + False Positives}}$$.  You can think of precision like quality - a region proposal algorithm with a very high precision is very unlikely to return irrelevant regions, although it may miss some relevant regions.  Recall is equal to the algorithm's $$\frac{\text{True Positives}}{\text{True Positives + False Negatives}}$$.  You can think of recall like quantity - a region proposal algorithm with a very high recall is likely to not miss any relevant regions but may also return some irrelevant regions.  For region proposal algorithms, we want to select algorithms and parameters that lead to a high *recall* - we're willing to waste some time checking irrelevant regions if that ensures that we won't miss any relevant regions.  

Region proposal algorithms can be evaluated by themselves by computing the recall between the proposed regions of interest to the actual bounding boxes of a dataset, with the labels being ignored.  Remember, we just need the region proposal algorithm to do a good job drawing boxes around possible objects.  
 
2 of the fastest and best performing algorithms for region proposal are selective search and edge boxes.  Although, these hard-coded region proposal algorithms are replaced with a learned region proposal network in the SOTA Faster-RCNN models, they are still quite interesting in how they do a decent job on a difficult task using a combination of simpler algorithms.  They're also nice in how they are dataset agnostic - edge boxes and selective search can do a good job proposing regions of interest regardless of whether they've seen a particular object before.

#### Selective Search
