# Deploying a Pipeline for a Model that uses a Custom Docker Image

In this exercise, you will create a machine learning pipeline for a  ML algorithm pipeline where a custom Docker image is now required. We are not using one of the standard Sagemaker images, but rather our own.

This pipeline is much simpler than our Customer Churn Step Function pipeline, as we want to focus in this example, on how a Docker image is created as part of a pipeline.

It could easily be extended to add in more steps if required.

We are also in this model introducing multiple CodeBuild steps.

Just like in the previous exercise, your challenge is to get this pipeline working for yourself!


# Your Tasks
Broadly, you will need to complete the following tasks.

1. Get the project into your own Git repository.
2. Set up a CodeBuild Project that will build your Docker image for you.
3. Set up a CodeBuild Project that will build the Model. 
4. Set up the necessary roles and permissions to allow the above to run successfully.
5. Review the Python script that sets everything up. Just like before, a number of updates will be required.
6. Set up the CodePipeline and CodeDeploy to run the Cloudformation template to deploy your application.


## 1. Get the Project into your own Git Repository

1. The repository URL for the example can be found here: https://github.com/nicktodd/machinelearning-custom-image

2. Fork this repository into your own Git repository. Your repository needs to be GitHub or CodeCommit as these are supported by CodePipeline and CodeBuild. We suggest you use GitHub since you will be able to authenticate more easily due to company restrictions on Code Commit.


## 2. Review the buildspec.yml, Dockerfile, and ML Script

1. Using your preferred editor, open the `buildspec.yml` file in your new Git project.

2.  Note the various sections. In the first section, we are logging into the Docker registry for our own ECR registry, and then also for the ECR registry of the location of our parent Docker image.

```
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin xxx.dkr.ecr.eu-west-1.amazonaws.com 
      - aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin 520713654638.dkr.ecr.eu-west-1.amazonaws.com
```

3. Review the commands in the build section. We are building a Docker image from the Dockerfile.

```
 build:
    commands:
      - account=$(aws sts get-caller-identity --query Account --output text)      
      - docker build -t custommlimage:latest .
      - docker tag custommlimage:latest ${account}.dkr.ecr.eu-west-1.amazonaws.com/custommlimage:latest
```

4. You must change the image name in the file to be something unique to you. You could simply add your initials on the end of the image name and tag. In simple terms, change every reference to `customimage` to something else, eg. `customimage-nt`.

Note the commands we are using to obtain our account ID. The service being used is the Secure Token Service, STS.

4. Now review the post_build phase. In there we are pushing our image up to the Elastic Container Registry ECR.

```
post_build:
    commands:
      - docker push ${account}.dkr.ecr.eu-west-1.amazonaws.com/custommlimage:latest
```
5. You will also need to change the name here from `customimage` to your new name. So make the change now and save the file.

6. Now open the Dockerfile. We don't need to spend a long time on this, but review the core sections. You can see that the parent image is a basic Ubuntu image! Nothing will be available, so in our RUN commands we are installing both Python and also the required Python libraries. Lastly, we are copying our ML script into the image (\coe\main.py). It is really only a fake script as you can see if you review it, but our focus is not the algorithm.


## 3. Set up the CodeBuild Project

6. Using the `AWS Web Console`, navigate to the `CodeBuild` service.

7. Click `Create Build Project`. Set the name to be [YourInitials]-CustomImageBuild. for the Source, link the project to your Git repository that you created earlier in step 1.

8. For the Environment, this is where you select the Docker image that will be used to complete your build. We just need a standard Amazon Linux for x86 processors.

9. Select that you would like to create a Service role.

10. In the `Privileged` section, tick the option to enable elavated privileges since we do want to build a Docker image. 

11. For the Buildspec, you can use the default which is to use a buildspec.yml file in the root of the project. That file is already there.

12. The rest can be left as defaults, so you can simply select `Create build project`.


## 4. Define the relevant IAM policies and roles

1. Navigate to IAM and locate the role that you just created, and then locate the policy. You will need to check that it has the necessary access to ECR. Specifically it needs to be able to push images and get the login password.

An example is located in[iam_policy_examples/docker_image_codebuild.json](iam_policy_examples/docker_image_codebuild.json).


## 5. Run the CodeBuild Project

1. Commit and push your changes to Git, and this should trigger your build. Check that it behaves as expected.

2. If it works, you will see a new image in your ECR repository. YOu can find it by navigating to the Elastic Container Registry it in the Amazon Web Console.


## Create a Second CodeBuild Project to complete the machine learning

So you now have a CodeBuild step that will create the Docker image. What we will now do is create another CodeBuild step that will do the machine learning. It could be done all in one, but this separation gives us more flexibility when running our actions.

1. Create a new CodeBuild project called `CustomImageMachineLearn[YourInitials]`.

2. Go through the steps linking it to your Source repository as before, and then for the buildspec file specify a different name this time, as the buildspec is now called `buildspec-model.yml`.

3. In your preferred code editor, open `buildspec-model.yml`. In this example, we are not using Step Functions (although we could if wanted to), but rather we are simply creating the model using the SageMaker API. 

4. Locate the Role name variable in line 10 and replace it with your AmazonSagemaker execution role (there is probably one already in your account).

5. On line 15, update the prefix variable to also include your initials at the end. eg:

```
prefix = 'custom-ml-image-nt'
```

6. On line 21, replace the image name with your chosen image name from your build step from earlier on.

Finally, review the rest of the file. It should look familiar as it is using the SageMaker API to create a model based on learning from a dummy CSV file.

You will also note that it creates a parameter file, similar to the previous example ready for deployment by Cloudformation.

7. Commit your changes to Git, and review the project execution in CodeBuild. Fix any errors before proceeding.

## Create the CodePipeline along with the CloudFormation CodeDeploy step.

The process is much the same as the previous Customer Church example. However our build step will involve two Code Builds instead of one.

The pipeline should look like the below:

![Custom Image Pipeline](images/custom_image_pipeline.png)

You can set this pipeline up for yourself using your experience from the previous pipeline to remind you how to do it. We recommend using the Create Pipeline wizard as it is the least painless option. A few things to bear in mind:

1. You will have to go back into the pipeline to amend the Cloudformation parameter location as you did in the previous exercise as the BuildArtifact option will be missing
   
2. You will also have to add in the second Action Group after the pipeline has been created. Since when you use the wizard it assumes only one build step, but you will have two.

## Reviewing the Failing Cloudformation Template Deployment




## Troubleshooting

If there are errors, review the error messages either in the CodeBuild log, or in the CodePipeline log if the CodeBuild has worked correctly.

The CodeDeploy error messages are not always that helpful unfortunately, but the most likely cause or errors will be mistakes in your filenames for the Cloudformation template and parameters files, and ensuring that you have the correct permissions in place.

Using the Web console, go to the CloudFormation service and locate your pipeline. If it is not there, then CodeDeploy didn't even get as far as trying to deploy it. If it is there, if it worked it will be there as `CREATE_COMPLETE` in green, or if not, then it will be there in red as `ROLLBACK_COMPLETE`.

If you are unsure about your template, try deploying it directly through the Cloudformation service. You will have to pass in the various parameters in the Web console but it will work correctly. That can be a useful way to check whether you have a problem with the template or its parameters or with the CodeDeploy itself.