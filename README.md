<h1>Creating Lambda Functions Using the AWS SDK for Python</h1>

<h2>Lab overview</h2>

In this lab, I will use the AWS SDK for Python (boto3) to create AWS Lambda functions. Calls to the REST API that I created in the earlier Amazon API Gateway lab will initiate the functions. One of the Lambda functions will perform either an Amazon DynamoDB database table scan or an index scan. Another Lambda function will return a standard acknowledgment message that I will enhance later in a lab where I implement Amazon Cognito.

<h2>What I learned:</h2>
<ol>
      <li>Create a Lambda function that queries a DynamoDB database table.</li>
      <li>Grant sufficient permissions to a Lambda function so that it can read data from DynamoDB.</li>
      <li>Configure REST API methods to invoke Lambda functions using Amazon API Gateway.</li>
</ol>

<h2>Business scenario</h2>
The café is eager to launch a dynamic version of their website so that the website can access data stored in a database. Sofía has been making steady progress toward this goal.
In a previous lab, you played the role of Sofía and created a DynamoDB database. The database table contains café menu details, and an index holds menu items that are flagged as specials. Then, in another lab, you created an API to add the ability for the website to receive mock data through REST API calls.
In this lab, you will again play the role of Sofía. You will replace the mock endpoints with functional endpoints so that the web application can connect to the database. You will use Lambda to bridge the connection between the GET APIs and the data stored in DynamoDB. Finally, for the POST API call, Lambda will return an updated acknowledgment message. 

<h2>Task 1: Configuring the development environment</h2>
In this first task, you will configure your AWS Cloud9 environment so that you can create Lambda functions. 
Connect to the AWS Cloud9 integrated development environment (IDE). From the Services menu, search for and select Cloud9. Notice the existing IDE, which is named Cloud9 Instance. For that IDE, choose Open. The AWS Cloud9 IDE loads in a new browser tab. Download and extract the files that you will need for this lab.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/27ed2348-6d66-4f24-b8e7-03494d5f23de" />

In the terminal, run the following command: wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCDEV-2-91558/05-lab-lambda/code.zip -P /home/ec2-user/environment

<img width="959" alt="image" src="https://github.com/user-attachments/assets/9a0154d3-0d54-4675-b139-61ea7111a74e" />

The code.zip file is downloaded to the AWS Cloud9 instance. The file is listed in the left navigation pane. Extract the file: unzip code.zip

<img width="959" alt="image" src="https://github.com/user-attachments/assets/1839ac18-9a79-4bf3-8caf-0c6968c9821b" />

Run the script that re-creates the work that you completed in earlier labs into this AWS account.
Note: This script populates the Amazon Simple Storage Service (Amazon S3) bucket with the café website code and configures the bucket policy as you did in the Amazon S3 lab. The script also creates the DynamoDB table and populates it with data as you did in the DynamoDB lab. Finally, the script re-creates the REST API that you created in the API Gateway lab.

To set permissions on the script and then run it, run the following commands:

        chmod +x ./resources/setup.sh && ./resources/setup.sh

When prompted for an IP address, enter the IPv4 address that the internet uses to contact your computer. You can find this IP address from https://whatismyipaddress.com.
Note: The IPv4 address that you set will be used in the bucket policy. Only requests that originate from the IPv4 address that you identify will be allowed to load the website pages. Do not set it to 0.0.0.0, because the S3 bucket's block public access settings will block this address.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/21039bf0-83c9-4d10-abe0-3bb6d1b56503" />

To verify the SDK for Python is installed, run the following command in the AWS Cloud9 terminal:

        pip show boto3
<img width="521" alt="image" src="https://github.com/user-attachments/assets/dfeb7a61-1545-4b8c-9328-68135ef55971" />

Take a moment to see what resources the script created.
<ol>
      <li>Confirm that an S3 bucket is hosting the café website files:</li>
      <li>In the Your environments browser tab, browse to the Amazon S3 console, and choose the name of the bucket that was created.</li>
      <li>Choose index.html and copy the Object URL.</li>

</ol>

Load the URL in a new browser tab. The café website displays. Currently, the website is accessing the hard-coded menu data that is stored in S3 to display the menu information.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/14260298-9afa-4ac8-aeae-a6dd1bbf1f37" />

<img width="959" alt="image" src="https://github.com/user-attachments/assets/6a2bb761-782f-49d0-801a-70f27a51cdcf" />
<img width="959" alt="image" src="https://github.com/user-attachments/assets/0bb53a07-9241-4ab7-9386-e9c3a3a6b486" />

Confirm that DynamoDB has the menu data stored in a table: Browse to the DynamoDB console. Choose Tables and choose the FoodProducts table. Choose Explore table items and confirm that the table is populated with menu data.
<img width="959" alt="image" src="https://github.com/user-attachments/assets/c8126bea-770b-4cdc-882d-9d91974159b7" />

Choose View table details and then on the Indexes tab, confirm that an index named special_GSI was created.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/67c64c08-407b-47c0-9c5d-0c6b25824fee" />

Confirm that the ProductsApi REST API was defined in API Gateway:
<ol>
      <li>Browse to the API Gateway console.</li>
      <li>Choose the name of the ProductsApi API.</li>
      <li>The API has a GET method for /products and a GET method for /products/on_offer.</li>
      <li>Finally, the API has POST and OPTIONS methods for /create_report.</li>
<ol>
<img width="959" alt="image" src="https://github.com/user-attachments/assets/f6d46b86-d771-4969-b9cd-e2c4798b6dfd" />
  
<img width="959" alt="image" src="https://github.com/user-attachments/assets/ff2c60f5-c4a0-4e59-8ef0-49226b751c7e" />

Copy the invoke URL for the API to your clipboard. In the API Gateway console, in the left panel, choose Stages and then choose the prod stage. Note: If you see a warning that you do not have ListWebACLs and AssociateWebACL permissions, ignore the warning. Copy the Invoke URL value that displays at the top of the page. You will use this value in the next step.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/74ee3341-fea8-4ca3-99ba-a0265a2c63ed" />

Update the website's config.js file. In the AWS Cloud9 IDE browser tab, open resources/website/config.js. On line 2, replace null with the invoke URL value that you copied a moment ago. Also, be sure to surround the URL in double quotation marks.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/d05ad3db-3aa1-43bd-b8d9-775903b985b0" />

Save the change to the file. 

Update and then run the update_config.py script. Open python_3/update_config.py in the text editor. Replace the <FMI_1> placeholder with the name of your S3 bucket. Tip: Find the bucket name in the S3 console, or run the following command:
      aws s3 ls

<img width="766" alt="image" src="https://github.com/user-attachments/assets/15c67de0-2620-4993-8188-39b52257af6e" />


Notice that this script will upload the config.js file that you just edited to the S3 bucket. Save the change to the file. To run the script, run the following commands

      cd ~/environment/python_3
      python update_config.py

<img width="573" alt="image" src="https://github.com/user-attachments/assets/c0e34b3f-5930-40de-b564-331d5f2ffe83" />

Load the latest café webpage with the developer console view exposed. 
Observe the Browse Pastries section of the webpage. Notice that only one menu item now displays.
This indicates that you are no longer viewing hard-coded data. Instead, the website returns the mock data just as you configured it to do in the previous lab.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/792fd0ac-96c1-4ae3-8288-797144ff0e4f" />

Your AWS account is now configured as shown in the following diagram:
<img width="431" alt="image" src="https://github.com/user-attachments/assets/ce06d6e4-3a66-4511-aa82-f202ec96d871" />

By the end of this lab, you will have created Lambda functions that the API will invoke. At that point, your account resources and configurations will look like the following diagram: 
<img width="417" alt="image" src="https://github.com/user-attachments/assets/68d7f94e-8f4f-4c2a-8df4-001a64a9da72" />

<h2>Task 2: Creating a Lambda function to retrieve data from DynamoDB</h2>
The first Lambda function that you create will respond to any /products GET requests from the website. The function will replace the mock endpoint that you created for the products API resource in the previous lab. Observe and edit the Python code that you will use in the Lambda function. In the AWS Cloud9 file browser, browse to and open python_3/get_all_products_code.py.
Replace the <FMI_1> placeholder and the <FMI_2> placeholder with the proper values.
  
<img width="635" alt="image" src="https://github.com/user-attachments/assets/63895878-92dd-4746-918b-abfe0f1ef64e" />

      Notice that the code does the following:
<ol>
        <li>It creates a boto3 client to interact with the DynamoDB service.</li>
        <li>It reads the items out of the table and returns the menu data.</li>
        <li>It also scans the table index and filters for items that are in stock.</li>
        <li>Save the changes to the file.</li>
</ol>
Test the code locally in AWS Cloud9.
To ensure that you are in the correct folder, run the following command:
       
        cd ~/environment/python_3
To run the code locally in the AWS Cloud9 terminal, run the following command:
       
        python3 get_all_products_code.py

If the code runs successfully, the first part of the table data is returned in the terminal output, formatted as a JSON document as shown here. 

<img width="959" alt="image" src="https://github.com/user-attachments/assets/cc666a62-cd6e-4581-be14-f0a7e72734dc" />

Modify a setting in the code and test it again.

In the get_all_products_code.py file, line 12 has the following code:
      
      if offer_path_str is not None:
      
Analysis: If the offer_path_str variable is not found, the condition fails and runs a scan of the table.
      To verify that this logic is working, temporarily reverse this condition. Remove the word not from this line of code, so that it looks like the following:
      
      if offer_path_str is None:

<img width="504" alt="image" src="https://github.com/user-attachments/assets/e8f4d5d6-109d-4fc0-92e0-1e8561a2bd4d" />

  Save the change. Run the code again:
<img width="839" alt="image" src="https://github.com/user-attachments/assets/9880b4ce-205a-4a65-8895-68b003655e0f" />

Notice that fewer menu items were returned this time. This was the intended result. The website calls to the REST API will implement the code so that customers can filter the menu for "on offer" menu items. Now that you know that the code logic works, reverse the code change and then define the Lambda function that will run the code. Comment out the last line of the code and save the file. 

<img width="563" alt="image" src="https://github.com/user-attachments/assets/bea511ce-7641-4f56-8d98-18762efe95df" />
Locate the IAM role that the Lambda function will use, and copy the role Amazon Resource Number (ARN) value. Browse to the IAM console, and choose Roles. In the search box search for and select the LambdaAccessToDynamoDB role that has been created for you.
Notice that this role provides read only access to DynamoDB. The policy provides enough access for Lambda to read the data that is stored in DynamoDB.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/d3ce1ee3-2e45-4348-b167-996f1051d209" />

Copy the Role ARN value.  You will use this in the next step. Edit the wrapper code that you will use to create the Lambda function.
<ol>
      <li>Return to the AWS Cloud9 file browser.</li>
      <li>Browse to and open python_3/get_all_products_wrapper.py.</li>
      <li>On line 5, replace <FMI_1> with the role ARN value that you copied.</li>
      <li>Save the changes to the file.</li>
<o/l>

<img width="715" alt="image" src="https://github.com/user-attachments/assets/1c61a63a-ef99-4ea9-8ed8-f65d7a70cb29" />

Observe what the get_all_products_wrapper.py code will accomplish when it is run:
          Creates a Lambda boto3 client.
<ol>
          <li>Sets the name of the role that the Lambda function should use.</li>
          <li>References the location of the S3 bucket that has the code that the Lambda function should run (the code you just zipped and placed in Amazon S3).</li>
          <li>Uses all of this information to create a Lambda function. The function definition identifies Python 3.8 as the runtime.</li>
</ol>
Package the code and store it in an S3 bucket.
<ol>
      <li>A bucket with -s3bucket- in the name was created for you when you started the lab.</li>
      <li>Verify that your AWS Cloud9 terminal is in the python_3 directory.</li>
</ol>

        cd ~/environment/python_3

To place a copy of your code in a .zip file, run the following command:

        zip get_all_products_code.zip get_all_products_code.py
        
Next, to retrieve the name of your S3 bucket, run the following command: 


























