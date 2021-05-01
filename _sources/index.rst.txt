.. Text Analytics Using Advanced Python documentation master file, created by
   sphinx-quickstart on Fri Apr 30 13:54:33 2021.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome to Text Analytics Using Advanced Python's documentation!
================================================================

Abstract
========

In this project I built a text analytics pipeline to efficiently process call transcript data on the local computer
system using dask inside luigi tasks. The purpose of this document is the technical specification and description of the
tool. Please note that this is not a user manual. This is an internal project of Mercedes-Benz Financial Service therefore
source code will not be part of this documentation.

To advanced python developer faster processing of data, which is the heart of the project, is a challenging task.
Besides the importance of good performance for large sets of data, there has
to be a flexible handling for file types and generating accurate model output that truely depicts reality on ground.
Distributing data processing to multiple cores of the machines has to be as smooth as possible. In order to modularize the code
we also need to robust pipeline that generally never fails and in rare event where it fails it
should start from where it left.

After reading this document you will know what design languages were used in implementing this pipeline and how it was developed.

Motivation
==========
In the customer obsession age we live in, the skill to predict customer needs in a fast and accurate
manner has become crucial to most businesses. However, while
customer focus products has become a matter of course, finding the right level of support and
anticipate customer needs still prove to be a day to day challenge.

A customer’s experience is very much a qualitative and emotion based experience. We are trying to turn this
into a quantitative measure. Whether it is Sentiment Score or Customer Satisfaction Score, we want to track a number.

Many businesses solely rely on this scoring system as they simply do not have time to do a more thorough analysis of the
feedback they are getting. That is where text analytics enters and creates the potential to gather insights
from millions of customer calls.

Any business that wants to provide a superior customer experience needs to utilise text analytics for mining voice of the
customer feedback and complaints. If you are not regularly assessing the qualitative data of what people like
or don’t like about your service then you will struggle to deliver that superior experience.

Text analytics models can turn the unstructured thoughts of customers into more structured data that can be
utilised by business. Without text analytics, it would take a huge amount of time and resources to analyse and
classify this data, never mind start to detect and score emotion sentiment.

Text analytics is truly capable of capturing the voice of the customer, finally allowing respondents to provide
feedback on their terms in their own voice, rather than simply selecting from a range of pre-set options.



Introduction
============

In 2020, Mercedes-Benz USA call centers handled 1.5 million inbound customer service calls. The company
wanted to utilize this data to understand the need of our customers. My team created a topic model that
analyzes transcripts and classifies them into a cluster of words associated with business processes. It serves the
purpose for a training team but has some shortcomings like:

* Memory error for training data of more than 60000 transcripts
* Lack CI/CD process making version controlling difficult
* Procedural pattern of code writing making code slow and inefficient (training time 2.5 hrs for one month of calls)
* Due to procedural design pattern it is difficult to modularize the code and thus DAG workflow is absent
* Model does not have an intuitive UI

Goal of this project is utilizing advanced python techniques to create a text analytics product capable of
analyzing large dataset locally and give insights on customer pain points.


Problem Scope
=============

* Here is a snippet of code which represents part of bottle neck in our old method:

.. figure:: https://raw.githubusercontent.com/nikhar1210/csci29_final_project/master/procedural1.png


Here we see how normalization is happening over loop, it inefficient to apply loop with compute intensive operations.
We also see after every transcript normalization we append it to a separate list, which makes whole process slow.

* Next example shows how replace on every transcript normalized in current model:

.. figure:: https://raw.githubusercontent.com/nikhar1210/csci29_final_project/master/procedural2.png


Here again we see similar pattern, where every iteration is checking particular word in transcript and replacing with a dictionary word.
This is happening while we are reading data from database thereby making whole system slow and difficult to distribute it across different cores.

* Finally in this next example we see how currently model displays output, it is an excel snapshot with topic cluster numbers and words with weigths associated with it in each cluster:

.. figure:: https://raw.githubusercontent.com/nikhar1210/csci29_final_project/master/procedural3.png

\

Therefore with these design choices we were destined for a poor performing pipeline.

\

Solution Design
================

1) Architecture Overview

The solution design follows the four prong approach.The first approach consists of the functional design pattern.
The second utilizes power of parallel compute using dask. The
third apply DAG workflow using luigi. Finally using power of django and t-SNE Corpus Visualization for model visualization

Here is the bird's eye view of full architecture:

.. figure:: https://raw.githubusercontent.com/nikhar1210/csci29_final_project/master/processflowarch.png

\

* Why not cloud?

One can argue why can't we harness the power of cloud and use infinite processing power to solve this problem. The reason is we can't
use NPPI sensitive data on cloud, Daimler AG is very sensitive about how customer data is used and therefore moving this to cloud will
be subjected to security internal audit review.


2) Existing Solution

Our model focuses on specific business language that revolves around our business. There are many business specific terms that are
used in our day to day operations, therefore the level of adjustments we can do in open source technology can not be done on products
already available in market like SAS callminer, IBM watson call assistant. With these reasons we
chose to develop in house solution.

3) Design Language

To create this model we relied on some key concepts discussed in class like :


* Functional Programming
   - Allows for faster execution
   - Easy to parallelize
   - Use of map partition

Here is an example from the project how functional programming allows us to
modularize our code for preprep of data and how map partition can apply a function
to various partition of data using dask.

.. figure:: https://raw.githubusercontent.com/nikhar1210/csci29_final_project/master/functional.png

\

* Decorators
   - Allows to minimize code writing
   - Allows to add new functionality to exisiting one

Here in the given example we are using luigi utils module's require decorator
to specify which task is required to be executed before a given class. It allows us
skip defining require function in every task thereby making code easier to read
without losing functionality.

.. figure:: https://raw.githubusercontent.com/nikhar1210/csci29_final_project/master/decorator.png

\

* Descriptors
   - Allows us to manage attributes of different classes
   - Allows us to get set or delete attributes of class if needed

Here in the given example we see how luigi's require decorator that we used is
executed on the backend, under __call__ method it modifies task_that_requires by adding requires method.
If only one task is required, this single task is returned. Otherwise, list of tasks is returned

.. code-block:: python

   class requires(object):

       def __init__(self, *tasks_to_require):
           super(requires, self).__init__()
           if not tasks_to_require:
               raise TypeError("tasks_to_require cannot be empty")

           self.tasks_to_require = tasks_to_require

       def __call__(self, task_that_requires):
           task_that_requires = inherits(*self.tasks_to_require)(task_that_requires)

           # Modify task_that_requires by adding requires method.
           # If only one task is required, this single task is returned.
           # Otherwise, list of tasks is returned
           def requires(_self):
               return _self.clone_parent() if len(self.tasks_to_require) == 1 else _self.clone_parents()
           task_that_requires.requires = requires

           return task_that_requires

\

* Context Manager
   - Parallelize sklearns algorithm across all cores of machine
   - Atomic write of parquet files
   - Allows workers to compute in parallel the application of a function
     to many different arguments

Here we are using context manager after creating local client with dask
in backend. It uses workers to compute in parallel the application of dask function. The main functionality it
brings in addition to using the raw multiprocessing are:

   - An optional progress meter (Needs bokeh preinstalled)
   - Interruption of multiprocesses jobs with ctrl C
   - Ability to use shared memory efficiently with worker processes for large numpy-based datastructures.

.. figure:: https://raw.githubusercontent.com/nikhar1210/csci29_final_project/master/contextmanager.png

\

Solution Architecture
=====================

Given figure below displays a part of luigi workflow of solution architecture.

.. figure:: https://raw.githubusercontent.com/nikhar1210/csci29_final_project/master/solutionarch.png

Here you will find task and their respective category. You will also find what each
task is dependent to and what atomic target it generates.

I won't be able to go into details what each task exactly doing and how it is
doing due to IP violation. But, there is some brief commentary about each of these
processes under final_proj_model module. It is available under module index.

\

Output
=======

Here is model output in an html file.
We can see how easy is to interact with this model. We can scroll through the topics
and filter words by weight by adjusting lambda slider at the top right.

Each word is sorted by weight in selected cluster and also shows number occurrences throughout the
corpus. It was made using pyldavis library.

.. figure:: https://raw.githubusercontent.com/nikhar1210/csci29_final_project/master/modeloutput.gif

\

Conclusion
==========

The text analytics projects with big data are very vast, new and the challenging.
We overcame some of that challenges discussed in solution design part. It explains that the performance and
the progression details of the mentioned methods.
Maximum number of projects about text analytics are on social media with sentiment analysis.
To develop the data quality and the analysis results understanding the technique by which data can be preprocessed is essential.
Datasets we are dealing with is very large and we had to process it with local machines.

Hence, with these constraints we still managed to find a solution that can now process upto 1million
call transcripts in one training cycle. By harnessing the power of parallel compute and DAG workflow
we processed 6 billion words and analyzed 200k customer touchpoints with company.

We can now run full pipeline for 1 month of call transcripts in less than 30 mins compared to
2.5hrs previously. Plus we can now visualize model output and interact with it to get better insights.


Learning
========

We learned that how important it is to modularize your pipeline and execute each module with
high efficiency using advanced python techniques. If these concepts were not introduced to me
I would have thought this is the best I could do. But advanced python has allowed me to become
better solution provider.






.. toctree::
   :maxdepth: 2
   :caption: Contents:


   modules



Indices and tables
==================

* :ref:`genindex`
* :ref:`modindex`
* :ref:`search`
