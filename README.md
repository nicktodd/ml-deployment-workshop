# ml-deployment-workshop
# Machine Learning Workshop

Welcome to this repository where you will find the instructions for how to complete the extended lab exercises for this program.

The labs are not designed to be click through exercises where you simply follow the instructions, but rather, you will need to research and investigate how best to solve the various challenges described here.

## Environment - Cloud9

The labs will require the AWS Console access and you will need to be working with Git as well. 

To simplify development and command line tasks, we recommend you create a Cloud9 Development environment that you can use for the duration of the training. 

Setting up Cloud9 is a straightforward process. YOu can create your environment using [these instructions](cloud9.md).

## The Three Core Exercises

The core of these exercises is to complete the following:

1. Deploy an application that consists of a number of Lambda functions that uses some of the AWS high level AWS ML services. The instructions are found [here](01_video_transcode_pipeline.md).


2. Build a Machine Learning Deployment Pipeline for an ML model that uses the prebuilt SageMaker Algorithm xgboost. The instructions are found [here](02_xgboost_customerchurn_pipeline.md).

3. Build a Machine Learning Deployment Pipeline for an ML model that uses a custom Docker image for a custom algorithm. The instructions are found [here](03_custom_image_pipeline.md).




For each of these exercises, all the code you should need for your pipelines has been provided for you in the form of a Git Repository. Your task in a nutshell, is to get it running in your own AWS account.

Each of the exercises is not meant to be the definitive way that you must deploy models, but rather, to illustrate some of the best practices, services from AWS, and techniques that you can apply when deploying your own models in a production environment.

