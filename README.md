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
li>It also scans the table index and filters for items that are in stock.</li>
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
                  
                  aws s3 ls
<img width="748" alt="image" src="https://github.com/user-attachments/assets/e342e02a-d79a-4d74-83fa-ac48561d4e04" />

Finally, to place the .zip file in the bucket, run the following command. Replace <bucket-name> with the actual bucket name that you retrieved: 
                  
                  aws s3 cp get_all_products_code.zip  s3://<bucket-name>

Verify that the command succeeded.
The response looks like the following: upload: ./get_all_products_code.zip to s3://<bucket-name>/get_all_products_code.zip

<img width="679" alt="image" src="https://github.com/user-attachments/assets/1818df6d-9bd1-4487-9ef5-de251a51f6f8" />

To create the Lambda function, run the following command:

                  python3 get_all_products_wrapper.py
                  
The output of the command shows DONE, confirming that the code ran without errors

<img width="358" alt="image" src="https://github.com/user-attachments/assets/f792dd4e-f22c-4a31-bf3d-bbdcb8c40cea" />

Observe the function that you created and test it. NBrowse to the Lambda console. Choose the name of the get_all_products function that you just created

<img width="959" alt="image" src="https://github.com/user-attachments/assets/a5691f24-0257-4040-9016-57d8e929f2ea" />

In the Code source panel, open (double-click) the get_all_products_code.py file to display the code.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/db75b67e-9994-4fe4-9d57-6911ff0dd138" />

Choose Test. For Event name, enter Products Keep all of the other default test event values, and choose Save. The test event is saved. Choose Test again. A tab that shows the results of your test displays, with a response that shows the data returned from the DynamoDB table. 

<img width="959" alt="image" src="https://github.com/user-attachments/assets/47f0a367-f1c2-4f0d-9a87-7ce0c997ea94" />

Create a new test event that is called onOffer.
<ol>
<li>In the Code source panel, open the Test menu (choose the arrow icon), and choose Configure test event.</li>
<li>Choose Create new event.</li>
<li>For Event name, enter onOffer</li>
</ol>

In the code editor panel, replace the existing code with the following:

                  { "path": "on_offer"
                  }
            
<img width="941" alt="image" src="https://github.com/user-attachments/assets/9b637b65-b893-4817-88f0-dd184f39b447" />

Choose Save. Choose Test. This time, the results only display the items that are on offer and not out of stock. Scroll to the bottom of the test results to the function logs. You see the log message running scan on index. As you can see, your Lambda function is now successfully retrieving data from the DynamoDB table. You have also observed that this one Lambda function can be used to return all menu items or only the ones that are on offer.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/db796d0e-870f-484e-97b4-5fc7ff9d3ba1" />

Congratulations on achieving this milestone! The next step is to configure the REST API to invoke this Lambda function whenever anyone requests product data on the website. You will also need to find a way to pass the path variable to Lambda when customers choose to view only the on offer products.

<h2>Task 3: Configuring the REST API to invoke the Lambda function</h2>
First, you tested the code locally in AWS Cloud9 to ensure that it worked. Then, you deployed the code as a Lambda function and tested that it worked as deployed. In this task, you will configure the /products and /products/on_offer REST API functions to invoke the get_all_products Lambda function so that the code can be invoked from the café website. 
Test the existing GET /products resource.
<ol>
<li>Browse to the API Gateway console.</li>
<li>Choose the ProductsApi API, and choose the GET method for /products.</li>
<li>Notice on the right side of the page that the method is still accessing a "Mock Endpoint".</li>
<li>Choose Test, and then choose Test at the bottom of the page.</li>
<li>Verify that the Response Body correctly returns the mock data.</li>
</ol>

<img width="959" alt="image" src="https://github.com/user-attachments/assets/a252df96-2659-43e5-91a9-fab20417fb38" />

<img width="804" alt="image" src="https://github.com/user-attachments/assets/7f5fa47d-8056-4083-a9a9-6599054cccc8" />

Replace the mock endpoint with the Lambda function.
<ol>
<li>At the top of the page. Ensure that the GET method is still selected under /products.</li>
<li>Choose Integration Request and Edit:</li>
<li>Integration type: Lambda Function</li>
<li>Lambda Region: us-east-1<li>Lambda Function: get_all_products</li>
<li>Choose Save</li>
<li>Choose Save.</li>
</ol>

<img width="959" alt="image" src="https://github.com/user-attachments/assets/bade0489-7600-452c-aaa6-310073b9fa20" />

Notice on the right side of the page that the method is no longer calling a "Mock Endpoint". Instead, it is calling your Lambda function. 

<img width="932" alt="image" src="https://github.com/user-attachments/assets/42c8c26a-67ea-479b-9f0b-fc35316432ad" />

Test the /products GET API call one more time by selecting Test. The call returns output similar to the following:

<img width="959" alt="image" src="https://github.com/user-attachments/assets/969b96c7-00b5-4636-ab34-3ac3f42834b0" />

Analyze the results. When you switched the integration endpoint from the mock endpoint to the Lambda function, the response headers that permitted Cross-Origin Resource Sharing (CORS) were removed. If you scroll down past the Response Data, the Response Headers section now shows only the following:

<img width="791" alt="image" src="https://github.com/user-attachments/assets/831d8da4-748b-4da2-b0ca-bb5e001dc255" />

The response header does not contain the CORS information that you need because API Gateway resides in a different subdomain (us-east-1.amazonaws.com) than the S3 bucket (s3.amazonaws.com). You could manually add the CORS configuration back to this resource using the AWS SDK. However, API Gateway has a feature that makes it simple to permit CORS.

Re-enable CORS on the /products API resource.
<ol>
<li>Choose /products so that it is highlighted.<l/i>
<li>Select the GET method. </li>
<li>Select Default 4XX and Default 5XX under Gateway responses. </li>
<li>Select GET under Access-Control-Allow-Methods. </li>
<li>Choose Save.</li>
</ol>

Using the same approach, update the /on_offer GET API method.
<ol>
<li>Choose the ProductsApi API, and choose the GET method for /on_offer.</li>
<li>Choose Integration Request and configure:</li>
<li>Integration type: Lambda Function</li>
<li>Lambda Region: us-east-1</li>
<li>Lambda Function: get_all_products</li>
<li>Choose Save.</li>
</ol>

<img width="950" alt="image" src="https://github.com/user-attachments/assets/a1ebbcf1-b5b0-4742-ac45-344be9d8dc8c" />

Test the /products GET API call one more time.
<ol>
<li>Choose the GET method for /products.</li>
<li>Choose Test, and then choose Test at the bottom of the page.</li>
<li>Scroll down to the Response Headers section again.</li>
<li>This time, Access-Control-Allow-Origin is included. </li>
</ol>
The following is an example of the output (your data will have a different value for Root):

<img width="928" alt="image" src="https://github.com/user-attachments/assets/d63f31cc-0634-4c64-8eeb-27caa7b79f86" />

<ol>
<li>Choose /on_offer so that it is highlighted.</li>
<li>Choose Enable CORS.</li>
<li>Select Default 4XX and Default 5XX under Gateway responses.</li>
<li>Select GET under Access-Control-Allow-Methods.</li>
<li>Choose Save.</li>
</ol>

<img width="959" alt="image" src="https://github.com/user-attachments/assets/64c109f6-9731-49cc-9399-661f60f6bcb1" />

<ol>
<li>Test the /on_offer GET API call..</li>
<li>Choose the GET method for /products/on_offer.</li>
<li>Choose Test, and then choose Test at the bottom of the page.</li>
<li>Scroll down to the Response Headers section. Notice that CORS is enabled in the headers.</li>
</ol>

<img width="770" alt="image" src="https://github.com/user-attachments/assets/0b551fd2-7e52-4140-b793-90c68b59c915" />
Scroll back up to the Response Body section. Do you notice an issue. The Lambda function is returning all of the menu item, not just the specials. This is because the conditional check for on_offer_str is not working.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/e5143c03-659e-4590-ad88-9a128c2f79fb" />
                  
                  offer_path_str = event.get('path')
                  if offer_path_str is not None:

<h4>Analysis: </h4>
You need to set the API logic for /products/on_offer to pass the path to the event object that Lambda uses. The next step explains how to do that.

Configure the /on_offer integration request details.
<ol>
<li>Choose the GET method for /products/on_offer.</li>
<li>Choose Integration Request.</li>
<li>Choose Edit. Expand Mapping Templates.</li>
<li>Choose Add mapping template.</li>
<li>In the Content-Type box, enter the following text:
application/json</li>
</ol>

Under Generate template choose Method request passthrough. Replace the text with the following:

      {
      "path": "$context.resourcePath"
        }
<img width="959" alt="image" src="https://github.com/user-attachments/assets/05f275d5-1f31-4be5-93c1-a3910f4bb289" />

This will evaluate to /products/on_offer. For now, the code simply checks for the existence of the variable. Choose Save at the bottom of the page. Test the GET method for /on_offer again. Choose Test, and then choose Test at the bottom of the page. Verify that only the on offer items (six items) are returned. Excellent! Now that this filter is working, you can deploy the API.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/becdbcb0-1093-40e0-a7a4-f54390f168e1" />

<h4>Deploy the API.</h4>
<ol>
<li>In the Resources panel, choose the API root /.</li>
<li>Choose Deploy API.</li>
<li>For Deployment stage, choose prod, and then choose Deploy.</li>
</ol>

<img width="959" alt="image" src="https://github.com/user-attachments/assets/d1cf6459-22f8-45f8-a154-a3b324e5a747" />
Congratulations! You have successfully updated the REST API so that it invokes a single Lambda function that can now filter for on offer items. With this logic, you do not need to create and maintain two separate Lambda functions to handle this functionality.

<h2>Task 4: Creating a Lambda function for report requests in the future</h2>
In this task, you will complete steps that are similar to what you just did. However, this time you will create the Lambda function for the /create_report POST action. Observe and test the Python code that you will use in the Lambda function. Back in the AWS Cloud9 IDE, browse to and open python_3/create_report_code.py.
Notice that this code does not do much yet. It simply returns a message. In a later lab, you will implement more useful logic to actually create a report; however, this code will suffice for now. Run the following command in the terminal:
     
      python_3/create_report_code.py

<img width="752" alt="image" src="https://github.com/user-attachments/assets/7d211df0-68e0-4be7-aee7-036902b6b2d2" />

Notice the capitalized "R" in the word Report. The mock data contained a lowercase "r" instead. This difference is how you can know that the website is accessing the Lambda function and not the mock data. Comment out the last line of the code in the create_report_code.py file. 
Save the change.

<img width="589" alt="image" src="https://github.com/user-attachments/assets/c9f56912-8a26-4980-baff-f27ff013a34f" />

Edit the wrapper code that you will use to create the Lambda function. Browse to and open python_3/create_report_wrapper.py. On line 5, replace the <FMI_1> placeholder with the LambdaAccessToDynamoDB Role ARN value. Tip: You may need to return to the IAM console to copy the Role ARN value. Save the changes to the file.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/f7bdb2ba-5610-4bcc-bd5e-8d92a4a39135" />

Package the code and store it in the S3 bucket. To place a copy of your code in a .zip file, run the following command:

        zip create_report_code.zip create_report_code.py
        
To place the .zip file in the bucket, run the following command. Replace <bucket-name> with the actual bucket name:

        aws s3 cp create_report_code.zip  s3://<bucket-name>
<img width="686" alt="image" src="https://github.com/user-attachments/assets/e4a1c30e-7673-4c98-983b-ff80552f3df8" />

Finally, to create the Lambda function, run the following command:

        python3 create_report_wrapper.py

The output of the command is DONE, confirming that the code ran without errors.
<img width="670" alt="image" src="https://github.com/user-attachments/assets/d27edb03-fd85-4f54-8717-c6cbff7eaf7e" />
Observe the create_report function that you created and test it.
<ol>
<li> Browse to the Lambda console.</li>
<li>Choose the name of the create_report function that you just created.</li>
<li>In the Code source panel, open (double-click) the create_report_code.py file to display the code.</li>
</ol>  

<img width="959" alt="image" src="https://github.com/user-attachments/assets/41da39ef-dd5c-4bce-bca2-d41566fb0466" />

In the Code source panel, open (double-click) the create_report_code.py file to display the code.
<ol>
<li> Choose Test. </li>
<li>For Event name, enter ReportTest</li>
<li> Keep all of the other default test event values, and choose Save.</li>
</ol>
The test event is saved. Choose Test again.
<img width="959" alt="image" src="https://github.com/user-attachments/assets/b67d59ff-be83-4752-99a8-25bb7ed4eafe" />

A tab that shows the results of your test displays, with a response that shows the message hardcoded into the function code. The following is an example: 
<img width="887" alt="image" src="https://github.com/user-attachments/assets/6dea50d0-747e-4530-94b2-00e94ac9873c" />

Note the capitalized "R" instead of the lowercase "r" that is used in the mock data. Therefore, you know that the website is accessing the Lambda function and not the mock data. 

<h2>Task 5: Configuring the REST API to invoke the Lambda function to handle reports</h2>

Recall that in a previous task you configured both GET methods in your API to invoke the first Lambda function that you created. In this task, you will complete similar steps to configure the POST method in the API to invoke the new create_report Lambda function. 

Test the existing POST method for /create_report Browse to the API Gateway console. Choose the ProductsApi API, and choose the POST method for create_report.
Notice on the right side of the page that the method is still accessing a "Mock Endpoint".

<img width="952" alt="image" src="https://github.com/user-attachments/assets/9cebedbd-5dd6-4da8-8f0d-78c1a19bf793" />

Choose Test, and then choose Test at the bottom of the page. Verify that the Response Body correctly returns the mock data (note the lowercase "r" in the results), as in the following example:
<img width="626" alt="image" src="https://github.com/user-attachments/assets/ad842c61-66cb-4624-b3d3-35fa744d6ccc" />

Replace the mock endpoint with the Lambda function.
<ol>
<li>Ensure that the POST method is still selected.</li>
<li>Choose Integration Request and Edit</li>
<li>Integration type: Lambda Function</li>
<li>Lambda Region: us-east-1</li>
<li>Lambda Function: create_report</li>
<li>Choose Save.</li>
</ol>

<img width="956" alt="image" src="https://github.com/user-attachments/assets/f64eeda8-1427-406a-ac07-13044bb5c226" />

Notice on the right side of the page that the POST method is no longer calling a "Mock Endpoint". Instead, it is calling the Lambda function. Test the method again.

<img width="959" alt="image" src="https://github.com/user-attachments/assets/5462744d-39ca-478f-bf93-251d312d22f0" />

Test the method again. The returned data looks like the following example. Note that you are now seeing a capitalized "R" instead of the lowercase "r" from the mock data:
<img width="947" alt="image" src="https://github.com/user-attachments/assets/85246534-cfd0-4b54-8d3a-55e25031c484" />

Deploy the API.
<ol>
<li>In the Resources panel, choose the API root /.</li>
<li>Choose Deploy API.</li>
<li>For Deployment stage, choose prod, and then choose Deploy.</li>
</ol>
Congratulations! You have successfully updated the REST API so that it invokes the create_report Lambda function.

<img width="956" alt="image" src="https://github.com/user-attachments/assets/7696d46e-fa51-45b2-813b-54dc63561865" />

<h2>Task 6: Testing the integration using the café website</h2>

In this final task, you will test both API calls (/products and /products/on_offer) through the website. 
Load the café website. Return to the browser tab where you have the café website open, and refresh the page. Note: To find the page again, browse to the Amazon S3 console. Choose the bucket name, choose index.html, and copy the Object URL value. Load the URL in a new browser tab. The café website displays. The website is now accessing the menu data that is stored in DynamoDB.
Test the menu items filter on the website. Scroll down to the Browse Pastries section of the page. By default, only the "on offer" menu items display. Choose view all and verify that more menu items are returned. If everything displays correctly, that means that your CORS configuration is working properly.

<img width="945" alt="image" src="https://github.com/user-attachments/assets/c9c73599-e1e6-434f-ab41-0591d7dc6741" />

Edit a menu item price in the DynamoDB table, and verify that the change is reflected on the website.
<ol>
<li>Select your favorite "on offer" menu item and note the current price.</li>
<li>In a different browser tab, go to the DynamoDB console and load the FoodProducts table items.</li>
<li>To open the menu item's record, choose the product_name hyperlink for that item.</li>
<li>Change the price_in_cents value to a different three-digit or four-digit number.</li>
<li>Save the change.</li>
<ol>

<img width="320" alt="image" src="https://github.com/user-attachments/assets/32671345-7b0a-4a3a-bde9-180ce9f1bd0a" />

<img width="959" alt="image" src="https://github.com/user-attachments/assets/52d96844-a437-415d-bd6f-e718d6cb3959" />

Reload the café website, and verify that the price change is reflected on the website.

<img width="284" alt="image" src="https://github.com/user-attachments/assets/c8c9adc9-df6e-4e32-b2cd-0301a1bcd098" />

<img width="427" alt="image" src="https://github.com/user-attachments/assets/2074cf24-8d82-4ced-8ced-b6cb67ac92d5" />

Sofía is satisfied that she has made progress!
Her plan to build a serverless, dynamic website with a database backend is really coming to fruition. Sofía's initial plan had three major milestones:
<ol>
<li> The first milestone was to create a database backend to store café data. She accomplished that in the DynamoDB lab.</li>
<li> The second milestone was to create a REST API so that the webpage hosted on Amazon S3 could interact with mock data. She completed the most difficult part of that task during the API            Gateway lab.</li>
<li>The third milestone was to create Lambda functions to query the DynamoDB table or index, or return a static message, and then to update the REST API resources to initiate the respective         Lambda functions.</li>
</ol>

You (acting as Sofía) have now built all of these resources in the labs up to this point in the course. The following diagram shows the architecture that you have built.

<img width="419" alt="image" src="https://github.com/user-attachments/assets/640a159a-1411-42df-9d3c-546361957504" />

Sofía knows that she has more work to do. For example, she needs to add authentication so that logged-in users can leverage the create_report API and generate reports. That will be implemented behind the login button that appears at the top of the website. Sofía also just heard from the café owners that they are in the final stages of acquiring another company (a coffee bean supplier). The acquisition will probably create additional requirements. For now though, Sofía decides to celebrate her most recent accomplishment by relaxing with her friends.

<img width="422" alt="image" src="https://github.com/user-attachments/assets/270d1e92-87ea-458a-a73d-886379202add" />

<h2>Lab complete</h2>

© 2024 Amazon Web Services, Inc. and its affiliates. All rights reserved. This work may not be reproduced or redistributed, in whole or in part, without prior written permission from Amazon Web Services, Inc. Commercial copying, lending, or selling is prohibited.








































































