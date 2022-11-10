# Creating a Basic Glue Job

## Introduction
In this exercise you will create a Glue job that will process and analyse some data using Spark.


## Part 1 Upload the Data

The first thing we need is some data! The data needs to be in S3.

1. Log in to your AWS account using your credentials and then ensure that you are in the region suggested by your instructor. The region is identified in the top right corner of the console.

1. In your AWS Account, enter ```S3``` in the *Services* search box, and then select **S3**.

2. In the S3 Service, select **Create Bucket**.

3. For the bucket name, enter ```glue-yourname```, substituting the text ```yourname``` with your actual name! Do not use any spaces or capital letters.

4. For the ***AWS Region***, select the region you selected at the start. Note that it will not be showing in the top right corner whilst in the S3 service.

5. Leave all the other settings as they are, and scroll to the bottom of the page and click **Create Bucket**.

6. The file we will be working with is a large set of Movie reviews. So using your Web browser, download the following file:
   
http://training.conygre.com/data/movies/labeledTrainData.tsv

7. Return to the AWS console, and at the list of buckets, select your bucket and then click **Upload**.

8. Select the file you downloaded in the previous step and then choose **Upload**. Do not leave the browser until you see a green bar at the top of the screen showing that the file has uploaded successfully.

## Part 2 Create the Glue Crawler

First we will create a Glue crawler that can identify the structure of the file for us.

1. In the **Services** search box of the **AWS Console**, type ```Glue```, hover over the **Glue** section that appears, and then under the **Top Features** that appear, select **Crawlers**.

2. In the Glue Crawler console, select the **Add crawler** button.

3. At the **Add information about your crawler** screen, enter the name ```YourNameMovieReviewCrawler``` substituting ```YourName``` for your actual name.

4. At the **Specify crawler source type** review the options, and without changing anthing, select **Next**.

5. At the **Add a data store**, leave the data store type as S3, and leave the Connection section blank. For the Include path, click on the folder icon, and then select your bucket. Leave the sample size blank, and click **Next**.

6. At the **Add another data store** click **Next**.

7. At the **Choose an IAM role** enter your name in the text box. This will create a role with the necessary permission to access your bucket. Click **Next**.

8. At the **Create a schedule for this crawler**, leave it as **Run on Demand**, and click **Next**.

9. At the **Add database** screen, enter ```yourname-glue``` for the database name substituting your actual name for ```yourname```. Leave the rest as it is and click **Next**.

10. At the **Configure the crawler's output** screen, click **Next** and then click **Finish**.

11. At the list of crawlers, select the checkbox next to your crawler, and click **Run crawler**.

12. After about 2-3 minutes you should see the status of your crawler change to **Stopping** and then **Ready**. At this point, from the menu on the left, select **Tables** and select the table called ```glue_yourname```.

13. You will see that the Glue crawler has successfully identified the 3 fields in the file called id, sentiment, and data type.

## Part 3 Set up the Glue Job

Now we can create a Glue job that can process the file. In our example, we will cleanse the movie review data so it could be used for some machine learning. What we will do is remove all the HTML and stop words and punctuation, plus convert the text into lower case. We will then write the modified file out to the bucket.

To create jobs you can use either a Notebook or Glue studio. For our example you will use a Notebook.

1. In the Glue menu on the left of the Glue console, under **Data Integration and ETL** select **Notebooks**.
   
2. At the Notebook Setup page, enter a name for your notebook as **YourNameGlueJobNotebook**. For the role, select the provided role called **GlueS3FullAccessRole**. Click **Start Notebook**.

3. You will now be presented with a Jupyter notebook embedded in the Glue Web console.

4. Scroll to the bottom of the Notebook and you will see some code something like the below:

```
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
  
sc = SparkContext.getOrCreate()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
```

You will recognise some of this code as being Spark code, but you can also see that there are some AWS specific libraries that extend the Spark platform. Most notably, the GlueContext which is wrapping the SparkContext. The GlueContext adds some additional capabilities to basic Spark.

5. Scroll back to the top of the notebook and you will see a Script tab. Note that from here, you can view the code without the rest of the notebook. This is ultimately what would be running as a Glue job.

6. Now click on the Job details tab.  Review the options here to see how you can configure the job when it runs.

7. Finally click on the Version control tab. You can see here how the code in the Job can be committed to a Git repository.

## Part 4 Implement the Glue Job

You wil now implement the required code to run a job that can process the movie data file.

We can read the data into a Dynamic frame. To create a Dynamic frame from data in a table in the glue catalog, the function takes 2 parameters
* The Glue database name
* The Glue table name

1. Add the following line of code to your end of your notebook substituting your database and table name.

```
glue_database = "YOURDATABASENAME"
glue_table = "YOURTABLENAME"
s3_write_location = "s3://YOURBUCKET/write"
dynamic_frame = glueContext.create_dynamic_frame.from_catalog(database = glue_database, table_name = glue_table)
```

2. If you want to use regular Spark APIs you can also read your dynamic frame into a normal Spark dataframe. So add the following line of code to do that. Also use the show() function to see what the file structure looks like.

```
spark_data_frame = dynamic_frame.toDF()
spark_data_frame.show()
```

You now have the dataframe in either a Spark or Dynamic frame format. 

We will now clean up  the text in the movie review column as if we were going to do some machine learning with it.

3. Run the code to check that it works using the play button, and then add the following import statement to the top of your notebook.

```
from pyspark.ml.feature import StopWordsRemover, RegexTokenizer
```

We can use these libraries to clean up the text.

4. Now go back to the end of the notebook. We will initially convert the text of the review into an array. You can achieve this with the following. Note the use of show() to see how it now looks. Run the code again to see the results.

```
regex = RegexTokenizer(pattern=r'(?:\p{Punct}|\s)+', inputCol='review', outputCol='review_as_array')
regex_df = regex.transform(spark_data_frame)
regex_df.show()
```

5. Now we can remove the stop words. These are words like *and, or, in, of* etc. Words that will not make much difference to a machine learning algorithm.

```
stopwords = StopWordsRemover(inputCol='review_as_array', outputCol='no_stop_words')
df_no_stopwords = stopwords.transform(regex_df)
df_no_stopwords.show()
```

Run the code again to see the results.

6. Now we will remove duplicates, and for that we can use the array_distinct function from the PySpark SQL functions library, so add the following import to the top of your notebook:

```
from pyspark.sql.functions import array_distinct
```

7. At the end of the notebook, add the following code:

```
df_no_duplicates = df_no_stopwords.withColumn("no_stop_words_or_dups", array_distinct("no_stop_words"))
df_no_duplicates.show()
```

8. There are a lot of columns now, so let's remove the intermmediate columns using drop(). We could have done this as we went along, but doing it this way means that you can see the process more clearly.

```
cleaned_df = df_no_duplicates.drop("review_as_array", "no_stop_words")
cleaned_df.show()
```

Finally, let's write the modified dataframe back to S3. To do this, we will convert the dataframe back to a Dynamic frame, and then write to S3. Add the following code to make these changes:

9. At the top of your notebook add an import statement for the DynamicFrame class:

```
from awsglue.dynamicframe import DynamicFrame
```

10. Now add the following code to write the dynamic frame to S3 using the S3 connector built into the DynamicFrame functionality.

```
dynamic_frame_write = DynamicFrame.fromDF(data_frame_aggregated, glueContext, "dynamic_frame_write")

#Write data back to S3
glueContext.write_dynamic_frame.from_options(
    frame = dynamic_frame_write,
    connection_type = "s3",
    connection_options = {
        "path": s3_write_path,
        #Here you could create S3 prefixes according to a values in specified columns
        #"partitionKeys": ["decade"]
    },
    format = "json"
)

```

11. Save your notebook using the Save button at the top of the browser, and then navigate to S3 and locate your bucket. You should now see a new file in there named something like ```run-1667927701481-part-r-00000```. Download the file and review it using VisualStudio Code or your preferred editor. You should see a JSON version of the movie data - ready for the next bit of processing!

# Part 5 Optional - Explore Scheduling your Glue Job

Now you have a working Glue job, you can schedule it to run when you require. 

1. Go back to the main Glue service page by typing *Glue* in the **Services** search box.
   
2. In the left pane, select **Jobs**.

3. Select the checkbox next to your job, and then **Actions**, and then select **Schedule Job**.

4. In the name, enter ```YOURNAME Monthly job run```.

5. In the **Frequency** box, select **Monthly**.

6. You can leave the other options as they are and click **Create Schedule**. Confusingly the screen doesn't close, but you have now scheduled the job.

7. Go back to the Glue main page again and click on **Monitoring**. This is where you can review your job runs and see what has executed.

8. Finally, return to the list of jobs and select your job. Click on the **Schedule** tab. Here you can see the schedule. You may need to refresh the browser.

9. In the schedule, you will see your job. Select the radio button on the top right of the scheduled item, and then **Actions** and then click **Delete Schedule**. This will remove it from the scheduled jobs. You may need to refresh the browser to verify that it has gone.