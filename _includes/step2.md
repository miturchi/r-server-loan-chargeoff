
Now you're ready to follow along with Debra as she creates the scripts needed for this solution. <span class="sql"> If you are using Visual Studio, you will see these file in the <code>Solution Explorer</code> tab on the right. In RStudio, the files can be found in the <code>Files</code> tab, also on the right. </span> 

<div class="hdi">The steps described below each create a function to perform their task.  The individual steps are described in more detail below.  The following scripts are then used to execute the steps.  
<ul><li>
<strong>loanchargeoff_main.R</strong> is used to define the data and directories and then run all of the steps to process data, perform feature engineering, training, and scoring.  
<p></p>
The default input for this script uses 100,000 loans for training models, and will split this into train and test data.  After running this script you will see data files in the <strong>/LoanChargeOff/dev/temp</strong> directory on your storage account.  Models are stored in the <strong>/LoanChargeOff/dev/model</strong> directory on your storage account. The Hive table <code>loanchargeoff_predictions</code> contains the 100,000 records with predictions (<code>Score</code>, <code>Probability</code>) created from the best model.
</li>
<li>
<strong>Copy_Dev2Prod.R</strong> copies the model information from the <strong>dev</strong> folder to the <strong>prod</strong> folder to be used for production.  This script must be executed once after <strong>loanchargeoff_main.R</strong> completes, before running <strong>loanchargeoff_scoring.R</strong>.  It can then be used again as desired to update the production model. 
<p></p>
After running this script models created during <strong>loanchargeoff_main.R</strong> are copied into the <strong>/var/RevoShare/<user>/LoanChargeOff/prod/model</strong> directory.
</li>
<li>
<strong>loanchargeoff_scoring.R</strong> uses the previously trained model and invokes the steps to process data, perform feature engineering and scoring.  Use this script after first executing <strong>loanchargeoff_main.R</strong> and <strong>Copy_Dev2Prod.R</strong>.
<p></p>
The input to this script defaults to 10,000 loans to be scored with the model in the <strong>prod</strong> directory. After running this script the Hive table <code>loanchargeoff_predictions</code> now contains the predictions.  
</li></ul>
</div>

Below is a summary of the individual steps used for this solution. 
<ol>
<li class="sql">  <strong>SQLR_connection.R</strong>: configures the compute context used in all the rest of the scripts. The connection string is pre-poplulated with the default values created for a VM from the Cortana Intelligence Gallery.  You must  change the values accordingly for your implementation if you are not using the default server (<code>localhost</code> represents a server on the same machine as the R code),  user (<code>rdemo</code>), and password (<code>D@tascience</code>).  If you are connecting to an Azure VM from a different machine, the server name can be found in the Azure Portal under the "Network interfaces" section - use the Public IP Address as the server name. The user and the password can be modified from the script <strong>createuser.sql</strong> </li>

<li>
The first few steps prepare the data for training.
</li>

<ul>
<li>  <strong>step1_get_training_testing_data.R</strong>: Read input data which contains all the history information for all the loans from HDFS. Extract training/testing data based on process date (paydate) from the input data. Save training/testing data in HDFS working directory </li>

<li>  <strong>step2_feature_engineering.R</strong>:  Here we use MicrosoftML to do feature selection. Code can be added in this file to create some new features based on existing features. Open source package such as Caret can also be used to do feature selection here. Best features are selected using AUC. </li>

<li>	<strong>step2_feature_engineering.R</strong>:  Performs Feature Engineering and creates the Analytical Dataset. Feature Engineering consists of creating new variables in the cleaned dataset.  <code>SMS_Count</code>, <code>Email_Count</code> and <code>Call_Count</code> are computed: they correspond to the number of times every customer was contacted through these three channels. It also computes <code>Previous_Channel</code>: for each communication with the <code>Lead</code>, it corresponds to the <code>Channel</code> that was used in the communication that preceded it (a NULL value is attributed to the first record of each Lead). Finally, an aggregation is performed at the Lead Level by keeping the latest record for each one. </li>
</ul>



    
<div class="alert alert-info" role="alert">
<div class="cig">
You can run these scripts if you wish, but you may also skip them if you want to get right to the modeling.  The data that these scripts create already exists in the SQL database.
<p/>
</div>
<div class=" hdi" >
To run all the scripts described above as well as those in the next few steps, open and execute the file <strong>loanchargeoff_main.R.</strong>
<p/>
</div>
In <span class="sql">both Visual Studio and</span> RStudio, there are multiple ways to execute the code from the R Script window.  The fastest way <span class="sql">for both IDEs</span> is to use Ctrl-Enter on a single line or a selection.  Learn more about  <span class="sql"><a href="http://microsoft.github.io/RTVS-docs/">R Tools for Visual Studio</a> or</span> <a href="https://www.rstudio.com/products/rstudio/features/">RStudio</a>.

</div>

<li class="sql"> If you are following along and you have modified any of the default values created by this solution package you will need to replace the connection string in the <strong>.R</strong> file with details of your login and database name.  
   
 <pre class="highlight"> 
connection_string <- "Driver=SQL Server;Server=localhost;Database=Campaign;UID=rdemo;PWD=D@tascience"
  </pre>      

<div class="alert alert-info sql" role="alert">
    Make sure there are no spaces around the "=" in the connection string - it will not work correctly when spaces are present
<p>
    If you are creating a new database by using these scripts, you must first create the database name in SSMS.  Once it exists it can be referenced in the connection string.  (Log into SSMS using the same username/password you supply in the connection string, or <code>rdemo</code>, <code>D@tascience</code> if you haven't changed the default values.)
    </p>
    </div>

    This connection string contains all the information necessary to connect to the SQL Server from inside the R session. As you can see in the script, this information is then used in the <code>RxInSqlServer()</code> command to setup a <code>sql</code> string.  The <code>sql</code> string is in turn used in the <code>rxSetComputeContext()</code> to execute code directly in-database.  You can see this in the <strong>sql_connection.R</strong> file:

<pre class="highlight">
connection_string <- "Driver=SQL Server;Server=localhost;Database=Campaign;UID=rdemo;PWD=D@tascience"
sql <- RxInSqlServer(connectionString = connection_string)
rxSetComputeContext(sql)
 </pre>     

 </li>   
 <li class="sql">  After running the step1 and step2 scripts, Debra goes to SQL Server Management Studio to log in and view the results of feature engineering by running the following query:
        
<pre class="highlight">
SELECT TOP 1000 [Lead_Id]
    ,[Sms_Count]
    ,[Email_Count]
    ,[Call_Count]
    ,[Previous_Channel]
FROM [Campaign].[dbo].[CM_AD]
</pre>
</li>

<li>  Now she is ready for training the models, using <strong>step3_training_evaluation.R</strong>.  This step will train two different models and evaluate each.  
<p></p>
   
   The R script draws the ROC or Receiver Operating Characteristic for each prediction model. It shows the performance of the model in terms of true positive rate and false positive rate, when the decision threshold varies. 
</p><p>
   The AUC is a number between 0 and 1.  It corresponds to the area under the ROC curve. It is a performance metric related to how good the model is at separating the two classes (converted clients vs. not converted), with a good choice of decision threshold separating between the predicted probabilities.  The closer the AUC is to 1, and the better the model is. Given that we are not looking for that optimal decision threshold, the AUC is more representative of the prediction performance than the Accuracy (which depends on the threshold). 
</p><p> 
   Debra will use the AUC to select the champion model to use in the next step.
</p></li>

<li> <strong>step4_prepare_new_data.R</strong> creates a new data which contains all the opened loans on a pay date which we do not know the status in next three month, the loans in this new data are not included in the training and testing dataset and have the same features as the loans used in training/testing dataset.
</li>

<li> <strong>step5_loan_prediction.R</strong> takes the new data created in the step4 and the champion model created in step3, output the predicted label and probability to be charge-off for each loan in next three months.
</li>

<li class="hdi">
After creating the model, Debra runs <strong>Copy_Dev2Prod.R</strong> to copy the model information from the <strong>dev</strong> folder to the <strong>prod</strong> folder, then runs <strong>loanchargeoff_scoring.R</strong> to create predictions for her new data. 
</li>
<li> Once all the above code has been executed, Debra will use PowerBI to visualize the recommendations created from her model. 

{% include pbix.md %}

She uses an ODBC connection to connect to the data, so that it will always show the most recently modeled and scored data.
  <img src="images/visualize.png"> 
  <div class="alert alert-info" role="alert">
  If you want to refresh data in your PowerBI Dashboard, make sure to <a href="Visualize_Results.html">follow these instructions</a> to setup and use an ODBC connection to the dashboard.
  </div>
</li>
<li>A summary of this process and all the files involved is described <a href="data-scientist.html">in more detail here</a>.
</li>
</ol>