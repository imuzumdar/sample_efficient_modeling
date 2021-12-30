# Ntropy: Task 2 Assignment  
  
This repo contains my submission for  Ntropy's take home assignment. I decided to work on task 2.  
  
## Problem Statement  
  
In most real-world applications, labelled data is scarce. Suppose you are given the Fashion-MNIST dataset (https://github.com/zalandoresearch/fashion-mnist), but without any labels in the training set. The labels are held in a database, which you may query to reveal the label of any particular image it contains.   
  
>Your task is to build a classifier to  90% accuracy on the test set, using the smallest number of queries to this database.  

You may use any combination of techniques you find suitable (supervised, self-supervised, unsupervised). However, using other datasets or pre-trained models is not allowed.   
  
## My Submission  
  
This repo contains three main notebooks which capture my work in tackling the given problem:  
  
1. **unsupervised_analyses** - Some unsupervised analyses using dimensionality reduction, clustering, and a simple encoder   
2. **model_baseline** - A baseline model trained on a random sample of data to set some key benchmarks and establish performance expectations  
3. **active_learning_with_regularization_and_augmentation** - The main modeling pipeline I built for this task   
  
In the following section, I'll go over my analysis in greater detail and explain the key decisions that led to my solution.   
  
## My Approach  
  
**TL;DR:** I initially performed some unsupervised analyses to better understand the data and see how well I could make classifications without any label information. While I wasn't able to solve this problem using entirely unsupervised methods, my findings from this work inspired me to implement an active learning framework in my final solution. Ultimately, my approach achieved 90.06% accuracy on the test set using only 9.3% of the training labels, outperforming a baseline model trained using 44.3% more data. 
 
### Unsupervised Methods  
  
In starting to solve this problem, I conducted some unsupervised analyses to gain a better feel for some of the underlying patterns in the data. I wanted to understand how much I could distinguish observations from different classes without incorporating any label information. I thought these findings would  give me a useful baseline and a clearer idea of how much label information I'd need to accomplish the task.  
  
First, I created a simple 2-D embedding model with UMAP, a nonlinear dimensionality reduction technique, and plotted some visualizations of the test set's embeddings. I chose UMAP over some other approaches to manifold learning, like T-SNE, because it's relatively fast for high-dimensional data and preserves inter-class structures in the latent space fairly well. In my visualizations, I noticed that UMAP was able to learn clusters for some classes, like bags and trousers, very effectively, but had trouble discerning examples from more closely related classes, such as shirts, pullovers, and coats. Even after I tried increasing the dimension of the latent   
representations, I wasn't able to find a clear separation boundary between those classes. When ran DBSCAN on the test set embeddings, I achieved a decent (but unexceptional) homogeneity score of .64, and found that many of the clusters contained a significant number of observations from 3 or more different classes (often, shirts, pullovers, and coats).  
  
I thought I could overcome this issue by applying parametric UMAP (basically an autoencoder trained using the UMAP loss function) to learn smarter embeddings, but I quickly ran into issues. I discovered the UMAP loss function is fairly expensive to compute (even with several built-in optimizations), so training a neural network encoder on the full data was computationally infeasible in 4 hours. Because of this inefficiency, I had to train the parametric model on a subset  
of the train data (10,000 observations). The new embeddings of the test set were nearly identical to the original ones - with slightly more loose clustering - and did not add much meaningful value to my analysis.   
  
Although my unsupervised methods did not ultimately pan out as a final solution, they did help my better understand the data and inspire some ideas for solving this task. I knew that I'd be able to pretty easy detect certain items, like bags and trousers, with a few examples of each item type. Conversely, I knew that predicting pullovers and coats may require more data, carefully selected near the separation boundary. Rather than trying to hand-pick those observations myself, I thought I'd let my model decide which examples were most ambiguous to label. Enter: active learning.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    

### Setting a Baseline

Before plunging into any work with active learning, I wanted to train a baseline model to compare the active learning pipeline against. Any good experiment has a control group, and I thought that establishing a baseline would help me answer two important questions:

1. Does active learning provide value in solving the given task?
2. In a rough sense, how much data do I need to surpass the target accuracy threshold? 

To that end, I randomly sampled 10,000 data points from the train set and fit a CNN with a few 2-D convolutions, max pooling layers, and dense connections, along with drop-out regularization and batch normalization (NOTE: I chose this model architecture based on the empirical performance of similar architectures on image datasets of similar dimension). 

After 40 epochs of training, I obtained an accuracy of 88.88% on the test set. At this point, I still wasn't sure whether active learning would be helpful or not, but I now knew that if it were useful, I should be able to obtain a similar (or higher) accuracy using far fewer than 10,000 observations.

### Building an Active Learner with Data Augmentation

As I mentioned in the **Unsupervised Methods** section, I concluded from my earlier analyses that letting the model pick the samples it finds most confusing may be useful in building an accurate model that is also label-efficient. This realization inspired me to choose an active learning framework for this problem. I hoped that active learning would help the model be more thoughtful about which labels to extract from the database, since I knew from the problem statement that retrieving labels was an expensive task.

While I was building out my active learning solution, I was also thinking about ways to increase the size of the training data without having to query the database for additional labels. I realized that I could leverage data augmentation and create new training samples by applying several image transformations (rotations, reflections, translations, cropping, etc.) to queried data points. Though augmented samples increase the model's training time (which likely has its own costs), they don't require the model to pay any costs associated with generating labels. Given the nature of this problem, that was a tradeoff I was willing to make. 

With those ideas in mind, here is the final modeling pipeline that I ended up using for my solution:

1. Generate initial random sample of data from the train set and fit it on the CNN. Here, I chose to collect an equal number of instances per class (200) so the model could better identify which classes are easily separable and which aren't. The remaining examples went into a sampling pool which the CNN could query.
2. Create new samples with data generator based on the initial random sample. I chose to generated one new image for each original one. 
3. Train the CNN on this initial sample and its generated examples.
4. Query for observations that model has most uncertainty in assigning a label. For the querying strategy, I chose to use [margin sampling](https://modal-python.readthedocs.io/en/latest/content/apireference/uncertainty.html) since I found that the CNN struggled with the highest and second highest class probabilities were close to each other. 
5. Generate new data samples from the queried data. Here, too, I generated one new data sample per data point.
6. Train CNN on queried data and new data samples. Remove the queried samples from the sampling pool.
7. Repeat steps 4-6 desired number of times. 

In my final model, I ran 9 rounds of queries and trained the CNN on 400 new labels during each round.  In doing so, **I obtained a classifier with 90.06% accuracy on the test set using 11,200 total training examples and 5,600 observations, or 9.3% of the training database. This model achieved better performance than the baseline and used 44.3% less labels during training.** 

Below are some interesting observations I made during this phase of analysis:

1. I noticed that the model performed better when I used fewer rounds of querying and requested more samples in each round. In other words, querying 200 data points 10 times seemed to be more effective than querying 100 data points 20 times. I'm not entirely sure why that happened, but I thought it was worth noting. One conjecture I had was that the decision boundaries between certain classes were fairly complex, so querying more data points allowed the CNN to better capture, in aggregate, the boundaries' behaviors during training. 
2. Although my final model required 5,600 labels for 90.06% test accuracy, I was able to obtain comparable (albeit slightly worse) model performance using far feature observations. For example,  during one experiment, the CNN achieved 88.1% accuracy using 3,600 labels. This result makes me think that I could have potentially hit the accuracy target using a smaller set of labels, but unfortunately I wasn't able to find such a collection of data points during my experiments. 
  
## Discussion  
  
Here are some of my main takeaways and thoughts from working on this assignment.  
  
### Things that Went Well  
  
1. **I was pretty happy with how fast I was able to set up each of my notebooks.** I certainly owe a lot of credit to the clean abstractions in Sklearn and Keras' APIs (having compatible operators from differnt libraries makes modeling so much easier), but the speed with which I was able to integrate new components into my workflows (an active learner, a data augmenter, an embedding model, etc.) allowed me to iterate faster and try more techniques to solve the problem.   
2. **I'm glad I chose an active learning approach.** Although it's certainly possible I could have achieved better results using other methods (ex: self-supervised learning), I think that applying active learning worked fairly well given the time constraints. I was able to test a variety of sampling methods, track the model's progress incrementally over several rounds of querying, and troubleshoot when the model made poor queries. From an engineering perspective, the modAL library was easy to understand (solid documentation and design) and compatible with sklearn APIs. 
3. **I think the notebooks are organized and documented well.** I'm not always the biggest fan of Jupyter notebooks, but I do love how easy it is to add custom plots and rich markdown annotations. Notebooks are great for telling a story, and I think mine do a good job capturing my overall process in completing this task. 
  
### Areas to Improve  
  
1. **I wish I set up better infrastructure for experimentation.** I didn't do much rigorous hyperparameter tuning and relied a fair bit (maybe too much) on intuition when choosing some of the hyperparameter values (ex: dropout rate, querying strategy, number of data points to query and generate on each training iteration, etc.). In the real world, I would've used a proper validation set and set up some kind of cross-validator to make more informed choices in selecting my final model.    
2. **I could have implemented more persistent logging.** I was pretty careful in analyzing the results of each individual pipeline run, but having well-tracked metrics across different trials might have given me a more holistic understanding of my model's performance. Plus, in a production setting, good logging would help with troubleshooting issues or degradations in the model (that's the hope at least).   
3. **I could have been more thoughtful about my data augmentation strategy.** I knew from my pipeline runs that adding augmented samples to the training set created consistent lift in the model's performance, but I don't really know what an optimal number of samples would be. Should I have trained the model on more synthetic samples? Could I have added more complexity to my data augmenter? How would these changes impact training time costs? I suppose the answers to these questions would vary for different problems, but I'm not satisfied with my answers to them. It's certainly possible that I could have used ever fewer samples by building a better data augmenter and training on more synthetic samples. 

### Techniques to Try (If I had More Time)

1. **Self-supervised learning** is a pretty popular technique these days for learning expressive functions using a small amount of labeled training data. When the I read the prompt for this task, SSL was one of the first ideas that came to mind (even before I saw it in the last sentence). I ended up choosing active learning based on my findings from the unsupervised analyses and my belief that active learning would be easier to troubleshoot, but I'm sure that SSL would yield strong results for this task. 
2. **Custom sampling functions** may have allowed me to be more sample-efficient in querying the training data. All the sampling methods I tried were localized to each observation, meaning they didn't take any global patterns in the sample space into account. Maybe applying some weighted measure to the sampler which adjusts for macroscopic structures in the class boundaries would help the active learner be query fewer samples. There's a lot of room to play here and I wish I explored more of this space. 
3. **Fitting an autoencoder** to the data may have helped improve downstream training time (far smaller data dimension) and remove noise in the data, but I unfortunately wasn't able to find a quick and effective method in time. 

## Conclusion   
Overall, I'm happy that I was able to hit the target test accuracy of 90% with only 9% of the data. I thought I was fairly efficient in my work and was able to derive important findings that allowed me to finish the task in a reasonable time frame. That said, with some thoughtful improvements and a bit more time, I think I could have achieve a higher accuracy with even fewer data points. 

For next time.