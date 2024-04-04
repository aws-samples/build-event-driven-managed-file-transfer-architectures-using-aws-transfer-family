# Automated File Processing for Transfer Family Connectors

This solution demonstrates how to architect secure and compliant outbound file transfers with AWS Transfer Family SFTP Connectors and PGP encryption.

This solution illustrates an event driven architecture for pre-processing, encrypting, and sending files to external partners over the SFTP protocol using [AWS Transfer Family](https://aws.amazon.com/aws-transfer-family/) and additional supporting services like [Amazon S3](https://aws.amazon.com/s3/), [AWS Step Functions](https://aws.amazon.com/step-functions/), [Amazon DynamoDB](https://aws.amazon.com/dynamodb/), [AWS Lambda](https://aws.amazon.com/lambda/), [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/), [Amazon EventBridge](https://aws.amazon.com/eventbridge/), [Amazon SNS](https://aws.amazon.com/sns/) and [Amazon SQS](https://aws.amazon.com/sqs/).

At the core of this architecture, we use a Step Functions state machine to execute the steps for processing and encrypting the files, before sending them to an external SFTP endpoint. The process starts when a file is uploaded in the landing S3 bucket with EventBridge enabled, executing the state.
The state machine executes the following steps:

-	Retrieve partner’s parameters from DynamoDB.
-	Execute custom process (for this post, we will transform a CSV file into JSON file).
-	Encrypt the processed file using the PGP public key stored in Secrets Manager.
-	Send the encrypted file to the partner by using a Transfer Family SFTP connector.
-	Verify the status of the file transfer.
-	Delete the original and processed files if the file transfer is successful.
-	If any steps fail execution, the state machine sends a notification to a SNS topic to notify the failure.


![image](https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/f004daec-a7ce-4f8d-b225-00a528af408a)


For this solution, we emulate the partner's SFTP server by creating a Transfer Family server and an S3 bucket to store the incoming files.

This solution will be broken down into the following sections:

1. Configure the Transfer Family server.
2. Configure the SFTP connector.
3. Configure partner parameters in DynamoDB.
4. Test end-to-end.


# Prerequisites

For this post, we provide an [AWS CloudFormation](https://aws.amazon.com/cloudformation/) template that deploys the following resources. Download the template [here](https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/blob/main/template.yaml).

When you deploy the CloudFormation template, provide connectors-pgp-blog as **Stack name** as we use this name to identify resources throughout the post.

**Once the stack is deployed, take note of the values in the output tab. You need this information later.**


# Solution walkthrough

For this walkthrough, we will go through the following steps:

1. Configure Transfer Family server

2. Configure Transfer Family SFTP connector

3. Configure partner's parameters in DynamoDB

4. Test the solution end-to-end

We will use [AWS CloudShell](https://aws.amazon.com/cloudshell/) for executing commands through the entire post; make sure to leave the browser's tab with CloudShell open. Note that CloudShell is not supported in all AWS Regions. See [documentation](https://docs.aws.amazon.com/cloudshell/latest/userguide/supported-aws-regions.html) for details.

## Step 1: Configure Transfer Family server

This is a two-step process; first we generate a SSH key pair, and then we import the public key in the user named **testuser** in the Transfer Family server created by the CloudFormation template.

### Step 1.1: Generate the SSH key pair

Transfer Family accepts RSA-, ECDSA-, and ED25519-formatted keys; we use an RSA 4096-bit key pair.

- In CloudShell, generate a SSH key pair by running the following command:

  `ssh-keygen -t rsa -b 4096 -m PEM -f partner_01`

- When prompted to enter a passphrase, do not type anything and select Enter twice.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/6132fc41-6a46-4a49-a92c-79283f034f06">


### Step 1.2: Import the SSH public key in Transfer Family

- View the SSH public key by running the following command:

  `cat partner_01.pub`

- Copy the output of the command and paste it in **SSH public key** for the service-managed user.
- Open the [Transfer Family console](https://console.aws.amazon.com/transfer/), then select **Servers** from the navigation pane.
- On the Servers page, select the Server ID for server that was created by the CloudFormation template.
- Select user **testuser** and in the SSH public keys pane, choose **Add SSH public key**.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/8030668b-1258-4a8b-84ed-f3ab483e66cd">




- Paste the text of the public key you copied before into the **SSH public key** text box, and then choose **Add key**.



<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/0dd73720-b49e-42b6-aa85-2c0d4bcc3312">



## Step 2: Configure the SFTP connector

When you create a Transfer Family SFTP connector, you will provide the following configuration parameters:

- The credentials to authenticate to the remote SFTP server.
- The URL of the SFTP server you want to connect to (you should have the URL noted in your text editor).
- The ARN of the IAM role that Transfer Family assumes (you should have the URL noted in your text editor).
- The public portion of the host key that identifies the external server.

Additionally, we need to store the PGP public key to encrypt files before sending them using the SFTP connector. In the next steps, we generate the PGP key pairs and store the public key AWS Secrets Manager along with the credentials to authenticate to the remote SFTP server.


### Step 2.1: Generate the PGP key pair

In a real-world scenario, your business partner would provide you with their public PGP key that you would use to encrypt the files before sending it over. For this post, you generate a PGP key pair consisting of public key and private key. You will use the public key to encrypt the files before sending them to the SFTP server.

- Install GPG by running the following command in CloudShell:

  `sudo dnf install --allowerasing gnupg2-full -y`

- Generate a key pair by executing the following command:

  `gpg --full-generate-key`

You are prompted to enter certain specifications for your key pair.

- When prompted to "Please select what kind of key you want" type '1' and then "Enter", which selects option 1 RSA. As a reminder, RSA is a public-key encryption system that encrypts data with asymmetric encryption.

- For this post, we accept the default key size of 3072 bits. When prompted with "What keysize do you want?", press **Enter**, which causes 3072 bits to be chosen.

- Next you are asked "Key is valid for?" For this example, we hit **Enter** which means the key will never expire.

- Review the configuration and then enter **y** to confirm.

![image](https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/475521ac-91a3-4312-af35-a1a4bd06cd14)








Now, you are prompted to construct a user ID to identify your key pair. You must provide:

- **Real name**, we use **testuser** for this example.
- **Email**, enter the email you would like to be associated with this key pair. We use this email later when we encrypt a file. For this example, we use [testuser@example.com](mailto:testuser@example.com).
- Verify the information you entered and accept typing **O** for Okay.
- **Passphrase**, make sure you write down your passphrase, so you don't forget it for future use.

![image](https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/6ae6b450-f386-4224-a0a4-81d39cec9810)





When your key pair has been created, gpg outputs a **pub**, **uid**, and **sub**.


### Step 2.2: Export the PGP key pair

- Export the PGP public key to a file by running the following command:

  `gpg --output testuser-public-gpg --armor --export testuser@example.com`

PGP keys need to be formatted with embedded newline characters ("/n") in JSON format.

- Format the key by running the following command.

  `jq -sR . ./testuser-public-gpg`

- Copy the output of the command, we use it later.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/b7d9a105-91a3-4644-99b9-be233ba79d77">


### Step 2.3: Export the SSH private key

For this blog, we authenticate to the external SFTP server using private SSH key which also needs to be formatted with embedded newline characters ("/n") in JSON format.

- Format the private SSH key you generated at step 1.1 by running the following command:

  `jq -sR . ./partner_01`

- Copy the output of the command, we need it in the next step.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/9db0ac35-9a1b-4754-95d6-c86eb092ecd3">


### Step 2.4: Update the secret in AWS Secrets Manager

- Open the Secrets Manager console, in the left navigation pane, choose **Secrets**, and then select the secret named **aws/transfer/connector-partner_01**.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/8061e8cb-a64b-48c1-8775-06a3a62e17ac">


- In the **Overview** tab, choose **Retrieve secret value**.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/e2823979-1c0c-40fa-95d9-a5b18b871cf8">


- Select **Edit** and then **Plaintext**.

**NOTE: You must edit the secret via the Plaintext method, as the Key/value method will not format the keys correctly.**

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/fbabf388-7893-4f50-8353-262686e3735a">


- Replace the part that says **"PASTE-SSH-PRIVATE-KEY-HERE"** with your SSH private key copied from the previous step 2.3.
- Replace where it says **"PASTE-PGP-PUBLIC-KEY-HERE"** with your PGP Public Key copied from the previous step 2.2.
- Finally, choose **Save** to update the secret.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/51451238-7991-4d87-885c-52b89eb21fd2">

Your secret should now look like this:

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/04d1ed64-7981-45ba-9394-1d5f9bac4c10">


### Step 2.5: Identify the trusted host key

- Retrieve the host key of the SFTP server created by the CloudFormation template by running the following command:

  `ssh-keyscan <server-endpoint>`

You should see an output like this.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/b64e5ace-5be9-4c9a-a8a6-231f73daae4c">


- Copy the SSH host key and paste it in your text editor. You need the key in the next step.

### Step 2.6: Create the SFTP connector

Now that we have all the necessary prerequisites, we can create the SFTP connector.

- Open the [Transfer Family console](https://console.aws.amazon.com/transfer/).

- In the left navigation pane, choose **Connectors**, then choose **Create connector**.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/3b8f312c-51d1-4525-aef1-dc838d540149">


- In the **Create connector** page, choose **SFTP** for the connector type, and then choose **Next**.

- In the **Connector configuration** section, provide the following information:

For the **URL** enter the server URL you noted in your text editor. It should look like **sftp://s-xxxxxxxx.server.transfer.<aws_region>.amazonaws.com**

For both **Access role** and **Logging role**, choose the IAM role named **connectors-pgp-blog -SFTPConnectorRole-xxx**.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/a080e28d-d85f-4bbb-909a-fda5c3dcdbb8">


- In the **SFTP Configuration** section, provide the following information:

For **Connector credentials**, from the dropdown list, choose the secret named **aws/transfer/connector-partner_01**

For **Trusted host keys**, paste in the public portion of the host key you retrieved earlier using the **ssh-keyscan** command.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/a5cda315-e74b-45ae-b4f8-44d04ff22fc6">


- Finally, choose **Create connector** to create the SFTP connector.

Take note of the **Connector ID**, you need it later.

If the connector is created successfully, a screen appears with a list of the assigned static IP addresses and a **Test connection** button. Use the button to test the configuration for your new connector.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/6c8aa7fa-5406-4707-8e46-f4fd3b5218fb">


You should see a window like this:

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/c252fb46-aa10-4a54-8757-16480ae10032">



## Step 3: Configure partner's parameters in DynamoDB

In the previous steps, you created all the necessary resources. In this step, you create an item in the DynamoDB table **connectors-pgp-blog-CustomFileProcessingTable-xxx** containing all the necessary parameters to define an outbound file transfer:

- partnerId –The ID of the partner you are sending the file to.
- lambdaARN – The ARN of the Lambda function for transforming CSV files into JSON.
- pgpSecret – The ARN of the secret where we stored the PGP public key.
- utputBucket – The name of the S3 bucket where we stage the file.
- connectorId – The ID of the SFTP connector we want to use to send the file via SFTP.

### Step 3.1: Create the item in Dynamo DB

- Open the [Dynamo DB console](https://console.aws.amazon.com/dynamodbv2/).

- In the left navigation pane, choose **Tables**, then select table **connectors-pgp-blog-CustomFileProcessingTable-xxx**.

- Choose **Explore table items**, then choose **Create item**.

- In the **Attributes** section, for **partnerId** enter **partner_01**, choose **Add new attribute** andthenchoose **String.**

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/e123027b-bafd-4ea1-9b39-c06a9bdb1171">


- For **Attribute name** type lambdaARN, and for **Value** enter the ARN of the Lambda function named **connectors-pgp-blog-CSVtoJSONLambdaFunction-xxx**. You should have noted this ARN in your text editor.

- Add a new string attribute with **Attribute name** pgpSecret and **Value** the ARN of the secret named **aws/transfer/connector-partner_01**. You should have noted this ARN in your text editor.

- Add a new string attribute with **Attribute name** outputBucket and **Value** **connectors-pgp-blog-outboundtransfers3bucket-xxx**. You should have noted this bucket's name in your text editor.

- Finally, add a new string attribute with **Attribute name** connectorId and **Value** the ID of the SFTP connector you created in step 2.5, and then choose **Create item** to store the parameters for partner_01.

<img width="75%" alt="image" src="https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/21152f81-2fa1-4913-9360-2c17a76b0d33">


## Step 4: Test end-to-end

To test the entire workflow, you will now upload a dataset in csv format to the S3 landing bucket, specifying the S3 prefix matching the partnerId in the DynamoDb table. For this example, the partnerId is **partner_01** and therefore the S3 prefix is **partner_01/**. On the landing bucket, EventBridge is enabled and will execute the state machine when the new object is created.

### Step 4.1: Set environment variables in CloudShell

- To test the workflow we use the [AWS CLI](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/index.html#cli-aws). To simplify the execution of the CLI commands, we set environment variables by running the following commands in CloudShell:

  `export STACK_NAME=connectors-pgp-blog`
  ```
  export LANDING_BUCKET=`aws cloudformation describe-stacks | jq -r --arg STACK_NAME "$STACK_NAME" '.Stacks[] | select(.StackName==$STACK_NAME) | .Outputs[] | select(.OutputKey=="LandingS3Bucket") | .OutputValue'`
  ```
  ```
  export OUTPUT_BUCKET=`aws cloudformation describe-stacks | jq -r --arg STACK_NAME "$STACK_NAME" '.Stacks[] | select(.StackName==$STACK_NAME) | .Outputs[] | select(.OutputKey=="OutboundTransferS3Bucket") | .OutputValue'`
  ```
  ```
  export SFTP_BUCKET=`aws cloudformation describe-stacks | jq -r --arg STACK_NAME "$STACK_NAME" '.Stacks[] | select(.StackName==$STACK_NAME) | .Outputs[] | select(.OutputKey=="SFTPServerS3Bucket") | .OutputValue'`
  ```
### Step 4.2: Create an example csv file

- Create a sample csv file by running the following command:

  `echo -e "City,State,Population\nSalt Lake City,Utah,1000000" > dataset.csv`

### Step 4.3: Upload the csv file to the landing S3 bucket

- Upload the sample csv file to the landing S3 bucket by running the following command:

  `aws s3api put-object --body dataset.csv --bucket $LANDING_BUCKET --key partner_01/dataset.csv`

## Step 4.4: Review the state machine execution

- Open the [Step Functions console](https://console.aws.amazon.com/states/) and select state machine **CustomFileProcessingStateMachine-xxx**.

- Select the latest execution and take some time to review the steps executed.

The last step executed by the state machine (Delete Originally Uploaded File) invokes the Lambda function **connectors-pgp-blog-DeleteFileLambdaFunction-xxx** which is responsible for deleting the original file **partner_01/dataset.csv** in the landing S3 bucket, and files **partner_01/dataset.json** and **partner_01/dataset.json.gpg** in the output S3 bucket.

- Confirm the file **partner_01/dataset.csv** was successfully deleted by running the following command:

  `aws s3api list-objects-v2 --bucket $LANDING_BUCKET`

You shouldn't see any objects.

- Now confirm files partner\_01/dataset.json and partner\_01/dataset.json.gpg were successfully deleted from the output bucket:

  `aws s3api list-objects-v2 --bucket $OUTPUT_BUCKET`

You shouldn't see any objects.

### Step 4.5: Review the S3 content at destination

- Confirm the file dataset.json.gpg was received by the SFTP server and stored in the S3 bucket used as backend storage by running the following command:

  `aws s3api list-objects-v2 --bucket $SFTP_BUCKET`

The output should be:
```
    {
    "Contents": [
    {
    "Key": "dataset.json.gpg",
    "LastModified": "2024-03-01T01:41:15+00:00",
    "ETag": "\"5b860964174bca41703e8885bcb35caa\"",
    "Size": 20508,
    "StorageClass": "STANDARD"
    }
    ],
    "RequestCharged": null
    }
```
-	Download the encrypted file to CloudShell by running the following command:

  `aws s3 cp s3://$SFTP_BUCKET/dataset.json.gpg .`

-	Decrypt the file by running the following command:

  `gpg --output dataset.json --decrypt dataset.json.gpg`

-	When prompted, enter the Passphrase you configured at step 2.1. The output should be:

![image](https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/39c78cca-7f51-40e1-83e2-860a9bfc819f)


-	Finally, verify the content of the decrypted file by running the following command:

  `cat dataset.json | jq '.'`

The output should be:

![image](https://github.com/aws-samples/automated-file-processing-for-transfer-family-connectors/assets/59907142/b54d02a6-9a7c-4b42-90ac-0a507bf9115d)





# Clean up

Following this solution, you created several components that may incur costs. To avoid future charges, remove the resources with the following steps:

- Delete the S3 buckets content. Use caution in this step; unless you are using versioning on your S3 bucket, deleting S3 objects cannot be undone.

- To delete the S3 buckets content you can run the following commands in CloudShell.

  `aws s3 rm --recursive s3://$LANDING_BUCKET`
  
  `aws s3 rm --recursive s3://$OUTPUT_BUCKET`
  
  `aws s3 rm --recursive s3://$SFTP_BUCKET`

- Delete the SFTP Connector you created in Step 2.6.

- Delete the CloudFormation stack connectors-pgp-blog.

# Conclusion

In this solution, we walked you through how to create an event driven architecture for pre-processing, encrypting, and sending files to external partners over the SFTP protocol.

This solution enables you to meet your data security needs, encrypting files containing sensitive and/or regulated datasets using Pretty Good Privacy (PGP) before easily send these files to partner hosted remote SFTP servers, using AWS Transfer Family's fully managed SFTP connectors. To learn more about Transfer Family, visit our [documentation](https://docs.aws.amazon.com/transfer/) and [product page](https://aws.amazon.com/aws-transfer-family/).



## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

















