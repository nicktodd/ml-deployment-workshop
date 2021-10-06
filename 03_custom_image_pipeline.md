# Deploying a Pipeline for a Model that uses a Custom Docker Image

In this exercise, you will create a machine learning pipeline for a  ML algorithm pipeline where a custom Docker image is now required. We are not using one of the standard Sagemaker images, but rather our own.

This pipeline is much simpler than our Customer Churn Step Function pipeline, as we want to focus in this example, on how a Docker image is created as part of a pipeline.

It could easily be extended to add in more steps if required, such as updating our DynamoDB database table acting as the model registry.

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

1. The repository URL for the example can be found here: https://github.com/nicktodd/custom-image-anomaly

2. Fork this repository into your own Git repository. Your repository needs to be GitHub or CodeCommit as these are supported by CodePipeline and CodeBuild. 


## 2. Review the buildspec.yml, Dockerfile, and ML Script

1. Using your preferred editor, open the `buildspec.yml` file in your new Git project.

2.  Note the various sections. In the first section, we are logging into the Docker registry for our own ECR registry. The parent image is a Ubuntu standard image in Dockerhub.

```
pre_build:
    commands:
      - account=$(aws sts get-caller-identity --query Account --output text)
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${account}.dkr.ecr.us-east-1.amazonaws.com 
```

3. Review the commands in the build section. We are building a Docker image from the Dockerfile. In this section, you can see that a variable is being used for the account number.

```
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...    
      - cd docker
      - docker build -t anomalyimage:latest .
      - docker tag anomalyimage:latest ${account}.dkr.ecr.us-east-1.amazonaws.com/anomalyimage:latest
```

4. You must change the image name in the file to be something unique to you. You could simply add your initials on the end of the image name and tag. In simple terms, change every reference to `anomalyimage` to something else, eg. `anomalyimage-nt`.

Note the commands we are using to obtain our account ID. The service being used is the Secure Token Service, STS.

4. Now review the post_build phase. In there we are pushing our image up to the Elastic Container Registry ECR.

```
post_build:
    commands:
      - docker push ${account}.dkr.ecr.us-east-1.amazonaws.com/anomalyimage:latest
```
5. You will also need to change the name here from `anomalyimage` to your new name. So make the change now and save the file.

6. Now open the Dockerfile. We don't need to spend a long time on this, but review the core sections. You can see that the parent image is a basic Ubuntu image! Nothing will be available, so in our RUN commands we are installing both Python and also the required Python libraries and various other required libraries. In addition, we are copying over the ML code into the image and giving it write permissions.

Note the presence of two files particularly:

* anomaly-model/train
* anomaly-model/serve

These are the two files that SageMaker will use when it runs the model. To do training, SageMaker runs `docker run image-name train` and when used for predictions it uses `docker run image-name serve`. 

If you are interested in the Algorithm and how it works, feel free to explore the various files located in the `anomaly-model` directory.

7. Finally, commit your changes to git using:

```
git add *
git commit -m "Updated image name"
git push
```

## 3. Create the Docker Registry

The script pushes the Docker image to a registry. This needs to be created so it can push successfully. 

1. In the Web console, to navigate to the Elastic Container Registry service, type in the search bar at the top `ecr` and the click on the `Elastic Container Registry`.

2. At the list of repositories, click the orange `Create Repository` button. 

3. At the `Repository name` field, enter the name you gave your Docker image tag, eg. `anomalyimage-nt`.

4. Leave everything else on the defaults and click `Create Repository`.


## 4. Set up the CodeBuild Project

6. Using the `AWS Web Console`, navigate to the `CodeBuild` service.

7. Click `Create Build Project`. Set the name to be `[YourInitials]-AnomalyImageBuild`. For the Source, link the project to your Git repository that you created earlier in step 1.

8. For the Environment, this is where you select the Docker image that will be used to complete your build. We just need a standard Amazon Linux for x86 processors.

9. Select that you would like to create a Service role.

10. In the `Privileged` section, tick the option to enable elavated privileges since we do want to build a Docker image. 

11. For the Buildspec, you can use the default which is to use a buildspec.yml file in the root of the project. That file is already there.

12. The rest can be left as defaults, so you can simply select `Create build project`.


## 5. Define the relevant IAM policies and roles

1. Whilst in your codebuild project, click on **Build Details** and then scroll down to the **Environment** section. Click on the link to the service role. Add the following additional policy - `DockerCodeBuildPolicy`. It has been created for you and will privde necessary access to ECR. Specifically it needs to be able to push images and get the login password.

If you would like to review the policy, you can see it in [iam_policy_examples/docker_image_codebuild.json](iam_policy_examples/docker_image_codebuild.json).


## 6. Run the CodeBuild Project

1. Commit and push your changes to Git, and if you are using GitHub this will trigger your build. If you are using CodeCommit, manually start a build. Check that succeeds.

2. If it works, you will see a new image in your ECR repository. YOu can find it by navigating to the Elastic Container Registry it in the Amazon Web Console.

3. If it fails, check the error log. The chances are you have either got your repository name wrong or you have not added the relevant permissions to your CodeBuild IAM role policy.

## 7. Create a Second CodeBuild Project to complete the Machine Learning

So you now have a CodeBuild step that will create the Docker image. What we will now do is create another CodeBuild step that will do the machine learning. It could be done all in one, but this separation gives us more flexibility when running our actions.

1. Create a new CodeBuild project called `CreateAnomalyModel[YourInitials]`.

2. Go through the steps linking it to your Source repository as before, and then for the buildspec file specify a different name this time, as the buildspec is now called `buildspec-model.yml`.

3. Click the **Create build project** button

4. In your preferred code editor (probably Cloud9), open `create-model.py`. In this example, we are not using Step Functions (although we could if wanted to), but rather we are simply creating the model using the SageMaker API. 

4. Locate the Role name variable in line 10 and replace it with the fully qualified role name of the `AmazonSagemakerExecutionRole` in your account (your can find it in the IAM service and search for roles).

5. On line 17, update the prefix variable to also include your initials just before the date stamp. eg:

```
prefix = 'anomaly-ml-image-nt' + dateAsString
```

6. On line 24, replace the image name with your chosen image name from your build step from earlier on.

You will see that an Estimator is created in order for us to do the Machine Learning. As it happens, this particular algorithm doesn't really need to do any learning, so actually not much is happening here! However, SageMaker requires a Model to be created so we have to do something. If you are interested, you can review the anomaly-model/train Python script to see what is actually happening.

You will also note that it creates a parameter file, similar to the previous example ready for deployment by Cloudformation.

7. Commit and push your changes to Git.

8. We now need to give our CodeBuild project the necessary permissions, so in the AWS Web Console, return to your new CodeBuild project. Click on `Build Details`, and then click on the `Service Role`. Add the following policies to the service role:  `S3FullAccess`, `SagemakerFullAccess`, and the `DockerCodeBuildPolicy`.

9. Run your build in CodeBuild and make sure it works. Fix any errors before proceeding.


## 8. Create the CodePipeline along with the CloudFormation CodeDeploy step.

The process is much the same as the previous Customer Churn example. However we will have two build stages instead of one.

When completed, the pipeline should look like the below:

![Custom Image Pipeline](images/custom_image_pipeline.png)

You can set this pipeline up for yourself using your experience from the previous pipeline to remind you how to do it. We recommend using the Create Pipeline wizard as it is the least painless option. A few things to bear in mind:

1. In the **AWS Web console**, go to the **CodePipeline** service. Click **Create Pipeline**.

2. Enter a name for your pipeline eg. `CustomImageAnomalyPipelineYOURINITIALS`.

3. Leave everything else as it is and then and click **Next**.

![Create Anomaly Pipeline](images/create-anomaly-pipeline.png)


4. At **Add Source Stage** screen, link it to your CodeCommit repository and select the master branch. Leave everything else as defaults and click **Next**.

5. At the **Build Provider** screen, select **AWS CodeBuild** and then select your Docker build CodeBuild project that you created earlier, leave everything else as default and click **Next**.

| :memo:        | Note that now you are looking at the **Add Deploy Stage** screen. Unfortunately, the wizard doesn't appreciate that you might actually want another build stage, so we will just have to add the Deploy stage and then edit the pipeline afterwards.  |
|---------------|:------------------------|

6. At the **Add Deploy Stage** screen, set the **Deploy provider** to be `AWS Cloudformation`. Set the **Action mode** to be `Create or Update`.

7. For the **Stack name**, enter `CustomImageAnomalyINITIALS`.

8. For the **Template name** enter select that it is a **Build Artifact** and enter the name of the file in your CodeCommit repository, which is `cloudformation/deploy-model.yml`. Note that it is NOT a build artifact. We will correct it later.

9. Enable the slider for the **Template configuration**. Set the artifact to be a **Build Artifact**, and for the name, enter `cloudformation_parameters.json`.

10. Set the Capabilities to include CAPABILITY_IAM and set the role to be  `CodeStarWorker-test-CloudFormation`.

The configuration should look something like this:

![Code Pipeline Initial Configuration](images/code-pipeline-anomaly.png)

11. Click **Next** and the Create the Pipeline.

12. Once the pipeline has been created, it will run automatically. We dp not want it to run because it is not complete. So click the **Stop Execution** button, then select your execution, and click **Stop and abandon**, and then click **Stop**.

## Modify the Pipeline to Incorporate the additional Build Step

1. Now whilst looking at your pipeline, click the **Edit** button found near the top of your screen.
   
2.  your pipeline, locate and click the **Add Stage** button found between your build and deploy stages.

3. In the **Add Stage** dialog enter the name `BuildAnomalyModel` and then click **Add Stage**.

4. In the new stage, click **Add action group**.

5. At the **Edit Action** dialog, enter the Action name as `BuildAnomalyModel`, set the **Action provider** as `AWS CodeBuild`. 

6. Set the **InputArtifacts** as `Source artifacts` (it doesn't actually need any but we have to select one).

6. For the Project name, select your existing CustomAnomalyBuildINITIALS project to be the one to use.

7. Set the **Output artifacts** to `BuildArtifact`. This is important as BuildArtifact is used by the CodeDeploy step.

![Model Creation CodeBuild Step](images/codebuild-config-model.png)

8. Click **Done**.

You have now added in the new build step. Now we will make a minor modification to the Deploy step to add in the fact that we also need Source artifacts to access the CloudFormation template from Git.

9. Click **Edit stage** located next to the Deploy stage.

10. Now click on the grey colored pencil shaped edit icon just under the Cloudformation action group.

11. Under the **Input artifacts**, click **Add** and then select the `SourceArtifact`.

12. Now locate the **Template configuration** section and change the **Artifact name** from `BuildArtifact` to `SourceArtifact`.

13. Click **Done**.

14. Click **Done** in the various edit stages, and then click to Save your changes to the pipeline.

## 9. Run the Pipeline and Review the Deployment

1. Now click **Release Change** to run your pipeline and see what happens. It should go through no problem.

2. Using the AWS Web Console, navigate to the SageMaker service, and then click on the Endpoints link on the left.

3. Locate your endpoint in the list and select it.

4. You will see your Endpoint is `In Service`.

![Endpoint](images/endpoint-inservice.png)

5. Scroll down and you will see the graphs. Locate the link to `View Logs`and click it.

6. Select the link to the `log stream`. You will see all the health checks returning a healthy HTTP 200 response.

![Log Events](images/log-events.png)


## Troubleshooting

If there are errors, review the error messages either in the CodeBuild log, or in the CodePipeline log if the CodeBuild has worked correctly.

The CodeDeploy error messages are not always that helpful unfortunately, but the most likely cause or errors will be mistakes in your filenames for the Cloudformation template and parameters files, and ensuring that you have the correct permissions in place.

Using the Web console, go to the CloudFormation service and locate your pipeline. If it is not there, then CodeDeploy didn't even get as far as trying to deploy it. If it is there, if it worked it will be there as `CREATE_COMPLETE` in green, or if not, then it will be there in red as `ROLLBACK_COMPLETE`.

If you are unsure about your template, try deploying it directly through the Cloudformation service. You will have to pass in the various parameters in the Web console but it will work correctly. That can be a useful way to check whether you have a problem with the template or its parameters or with the CodeDeploy itself.