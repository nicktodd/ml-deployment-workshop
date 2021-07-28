# Setting up Cloud9 

CLoud9 is the AWS development environment that integrates with CodeCommit, Lambda, and many other AWS services.

![Cloud9](images/cloud9.png)

It comes preinstalled with Git, the AWS CLI, Docker, SAM, and various other useful tools, and you can install others, since it is fundamentally a Linux EC2 with a Web based IDE installed in it.

You might want to use this for the exercises.

## Setting up Cloud9 for the Exercises

1. Using your browser, sign in to the `AWS Console`, and select the `eu-west-1` region. Browse to the Cloud9 service and set up a Cloud9 environment. For the training, for the instance type, either a t3.medium or a t3.large will be about right. The smaller instance types will struggle with Docker.

Once the environment has been created. You will be able to access it via a Web browser. 


## Increasing a Cloud 9 Disk Size

A Cloud9 environment only has 10GB Disk space which is not enough when working with Docker images. This has to be  ameneded after you have created your Cloud9 Environment.

1. Using the **Amazon Console**, navigate to the **Cloud9** service and locate your Cloud 9 environment, but **DO NOT** launch it.
   
2. At the top right of the console, select **View Details**.
   
3. In the **Environment Details** section, click on the link to **Go to instance**.

4. You will now be looking at the EC2 console and your instance will be there. 

5. If your instance is showing as **running**, then right click your instance and select **Stop instance**.   

6. Select the checbox next to your instance and in the tabs below, select the **Storage** tab.

7. Under the Storage tab locate and then link on the link to your **volume-id**.

8.  You will now be looking at your Volume in the EBS service, so in the **Actions** menu, select **Modify Volume** and in the presented dialog, change the size to 50gb.

![Modify Volume](images/modify-volume.png)

9. Return to your Cloud9 IDE interface and open the IDE. You will now have enough disk space for the Docker images to be downloaded as and when you need them.

Further information on setting up Cloud9 can be found here:

https://cicd-pipeline-cdk-eks-bluegreen.workshop.aws/en/intro/cloud9.html

