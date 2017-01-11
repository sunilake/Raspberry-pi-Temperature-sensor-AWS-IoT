# Control AC using Raspberry Pi, AWS IoT and LIRC... 

## Activity 1 and Activity 2
## Overview

 A simple raspberry pi and AWS IoT application to send notification to user on temperature change.  The temperature sensor continuously senses and sends data to Amazon’s DynamoDB (sensor readings data stored) over secure AWS IoT connection and when temperature goes below a certain threshold (a rule) a text message (SMS) will be sent to your mobile device using Amazon SNS notification service.
Note: This experiment is specifically for DHT11 temperature and humidity sensor.

Logical design of project
![Alt text](https://github.com/pdeolankar/Raspberry-pi-Temperature-sensor-AWS-IoT/blob/master/logicaldesignimage.PNG?raw=true ":")

### Requirements

Hardware
* Raspberry pi 2 (with Raspbian Jessie OS) + breadboard  
* DHT11 temperature and humidity sensor
* Wi-Fi dongle
* Jumper wires  
* HDMI cable     	    
  
Software installations
* Python 3 
And here's some code!

```javascript
sudo apt-get install python3	
```
Note:  Pip is included by default with Python 3.4 libraries.
* Mosquitto MQTT client software (version 1.4 onwards preferred). There is really easy tutorial for installation I found [here](http://www.instructables.com/id/Installing-MQTT-BrokerMosquitto-on-Raspberry-Pi/)
```javascript
wget http://repo.mosquitto.org/debian/mosquitto-repo.gpg.key
sudo apt-key add mosquitto-repo.gpg.key
cd /etc/apt/sources.list.d/
sudo wget http://repo.mosquitto.org/debian/mosquitto-jessie.list
sudo apt-get update
sudo apt-get install mosquitto-clients
```
* Paho Python client
```javascript
sudo  pip install paho-mqtt	
```
* Adafruit Python DHT Sensor Library

1)	Library for python to communicate with Raspberry pi gpio pins
Tutorial for Adafruit’s guide and library installation can be found  [here](https://learn.adafruit.com/adafruits-raspberry-pi-lesson-4-gpio-setup/configuring-gpio)
```javascript
sudo apt-get update
sudo apt-get install python-dev
sudo apt-get install python-rpi.gpio
```
2)	This is a python library for reading DHT series of temperature and humidity on raspberry pi. For referrence code is available [here](https://github.com/adafruit/Adafruit_Python_DHT)
```javascript
git clone https://github.com/adafruit/Adafruit_Python_DHT.git
cd Adafruit_Python_DHT
sudo apt-get update
sudo apt-get install build-essential python-dev python-openssl
sudo python setup.py install
```
And,
* Amazon Web Services AWS IoT

Register and login through AWS console  [here](https://aws.amazon.com/).
Create a free account (check free tier details for more information). Now you can access all free tier amazon web services.
Create a AWS Identity and Access Management (IAM) corresponding account.
Note: The ‘region’ selected for this experiment is ‘us-west-2’.
Also install and configure AWS CLI, the instructions can be found  [here](http://docs.aws.amazon.com/cli/latest/userguide/installing.html)

* Connecting Raspberry pi with DHT11 sensor 
Set DHT11 temperature and humidity sensor on raspberry pi and bread board. Sample connection diagram:
![Alt text](https://github.com/pdeolankar/Raspberry-pi-Temperature-sensor-AWS-IoT/blob/master/dhtrpiconnimage.PNG?raw=true ":")

Power up your raspberry pi, take free breadboard, dht11 sensor and some jumper wires. Use of 10K Resistors is preferred.

1) connect negative pin of sensor to pin 6 of pi

2) connect vcc pin of sensor to pin 2 of pi

3) connect signal pin of sensor to pin 7 of pi

Note: For this tutorial, no resistors are used.

Test: To check connection a sample program, 
Run
```javascript
sudo python testrun.py
```
Note: testrun.py file is in test-run folder. The sample output for above file can be found in output-testrun file.

The output should show temperature in degree Celsius and humidity percentage readings every 2 seconds iteratively.

* Amazon Web Service’s AWS IoT connection
Let us connect our Raspberry pi to AWS IoT service, for more information read  [here](https://aws.amazon.com/iot-platform/).

Traditionally we need,
“thing” created on AWS IoT console, here a thing is our raspberry pi
Generate certificates and connect them the “thing”. Three certificates are needed are root CA certificate, client’s private key and client certificate generated by AWS IoT (these certificates names generated should be similar but not exact to ca.pem.cert and private_key.pem and certificate.pem).

1) The AWS IoT service root CA is provided by Symantec and can be accessed from  [here](https://www.symantec.com/content/en/us/enterprise/verisign/roots/VeriSign-Class%203-Public-Primary-Certification-Authority-G5.pem),
 generates ca.pem.crt file

Thing creation and generation of certificates can be through AWS console online and through AWS CLI.
Here this will be done using OpenSSL,

2) Creating a private key an RSA Key 
```javascript
openssl genrsa 2048 > private_key.pem
```
3) Creating client certificate
```javascript
openssl req -new -key private-key.pem -out csr.pem
```
This command will need you to answer few questions here, this process is interactive. The fields to answer will be to add Country Name, State or Province, Locality or City, Company, Organizational Unit, Common Name, email, password, company name and few extra attributes. Remember the answers you provide here.
```javascript
aws iot create-certificate-from-csr --certificate-signing-request file://csr.pem
```
The client certificate generated will be certificate.pem file.
```javascript
aws iot describe-certificate --certificate-id <your_certificate_id>
```
Now, you have ca.pem.crt, private_key.pem, certificate.pem and csr.pem certificates for IoT devices to establish a secure connection to AWS IoT.

* Create a 'thing'

A ‘thing’ is a client device connected to AWS IoT and all these devices can be found in AWS IoT a thing registry.
Create a thing for experiment, name it anything you like, here (the thing) named ‘temperature-1’.
Execute this command and you will see a ‘thingArn’ and ‘thingName’ on screen, this means that a thing is created and you can check it on AWS IoT console.

1) Create AWS IoT policy and attach it to client certificate (generated earlier).
The policy document is in json format, allowiotoperations.json
```javascript
{
	"Version":"2012-10-17",
	"Statement":[
	{
	  "Effect":"Allow",
	  "Action":[
	      "iot:*"
	   ],
	   "Resource":[
		"*"
	   ]
	}
    ]
}

```
```javascript
 aws iot create-policy --policy-name "AllowIotOperationsPolicy" --policy-document file://allowiotoperations.json
```
On screen policyName, policyArn, policyDocument and policyVersionId will be displayed.
```javascript
 aws iot attach-principal-policy --principal "<certificate ARN>" --policy-name "AllowIotOperationsPolicy"
```
To attach your certificate with its policy to your thing
```javascript
 aws iot attach-thing-principal --thing-name "temperature-1" --principal "<your_certificate_ARN>"
```
A unique AWS account custom endpoint is used for establishing a secure connection to AWS IoT, get your endpoint information using this command,
```javascript
 aws iot describe-endpoint
```
aws iot account specific endpoint address will be displayed on screen, something like this
e.g.	“abc123d4efghi.iot.us-west-2.amazonaws.com”

To test your devices connection to AWS IoT
```javascript
 mosquitto_pub --cafile ca.pem.crt --cert certificate.pem --key private_key.pem -h abc123d4efghi.iot.us-west-2.amazonaws.com -p 8883 -q 1 -d -t topic1 -i temperature-1 -m "Helloooo"
```
On successful connection output like this can be seen,
```javascript
Client temperature-1 sending CONNECT
Client temperature-1 received CONNACK
Client temperature-1 sending PUBLISH (d0, q1, r0, m1, 'topic1', ... (8 bytes))
Client temperature-1 received PUBACK (Mid: 1)
Client temperature-1 sending DISCONNECT 
```
* Create a Dynamodb table and keys

Using AWS CLI,
```javascript
aws dynamodb create-table --table-name IOT-TempData --attribute-definitions AttributeName=key, AttributeType=S AttributeName=timestamp,AttributeType=S --key-schema AttributeName=key, KeyType=HASH AttributeName=timestamp,KeyType=RANGE --provisioned-throughput ReadCapacityUnits=1, WriteCapacityUnits=1
```
On success, the TableDescription will be displayed and make a note of TableArn generated.
Now, we need previously created AWS IAM account, AWS IoT will assume your IAM role to process and place messages in DynamoDB.

1) Create actions role
Make file iot-tempdata-actions-role.txt and add,
Using AWS CLI,
```javascript
{
	"Version":"2012-10-17",
	"Statement":[
	{
		"Sid":"",
		"Effect":"Allow",
		"Principal":{
			"Service":"iot.amazonaws.com"
		 },
		 "Action":"sts:AssumeRole"
	}
    ]
}

```
```javascript
 aws iam create-role --role-name "iot-tempdata-actions-role" --assume-role-policy-document file://iot-tempdata-actions-role.txt
```
On success, you will see Role details, make a note of Arn.
Make a policy to attach to this role,

In file iot-tempdata-ddb-insert-policy.txt add,
```javascript
 {
	"Version":"2012-10-17",
	"Statement":[
		{
		   "Effect":"Allow",
		   "Action":[
			"dynamodb:*"
		    ],
		    "Resource":[
			"arn:aws:dynamodb:us-west-2:123456789101:table/IOT-TempData"
		      ]
		}
	]
}
```
Note: The arn mentioned in Resource above is the TableArn generated while creating a dynamodb table.
```javascript
 aws iam create-policy --policy-name "iot-tempdata-ddb-insert-policy" --policy-document file://iot-tempdata-ddb-insert-policy.txt
```
```javascript
 aws iam attach-role-policy --role-name "iot-tempdata-actions-role" --policy-arn "arn:aws:iam::123456789101:policy/iot- tempdata -ddb-insert-policy"
```
2) Create a Topic Rule
Amazon web services allow to describe specific actions to be performed based on AWS IoT topic strings received from client devices. These messages are parsed through MQTT topic strings, 

Example: humidity topic string may look like this, 
topic/tempdata/humidity

Create a Topic rule to add new item to dynamodb on receiving a sensor read
Make iot-tempdata-topic-ddb-insert-rule.txt file and add,
```javascript
 {
  "sql":"SELECT * FROM 'topic/tempdata/temperature'",
  "ruleDisabled":false,
  "actions":[
	{
	  "dynamoDB":{
		"tableName":"IOT-TempData",
		"hashKeyField":"key",
		"hashKeyValue":"${clientId()}",
		"rangeKeyField":"timestamp",
		"rangeKeyValue":"${timestamp()}",
		"roleArn":"arn:aws:iam:: 123456789101:role/iot-tempdata-actions-role"
	    }
	}
     ]
}
```

```javascript
 aws iot create-topic-rule --rule-name "iot_tempdata_topic_ddb_insert_rule" --topic-rule-payload  file://iot-tempdata-topic-ddb-insert-rule.txt
```

* Amazon Web Service CloudWatch logs

CloudWatch logs displays performance measure of all operations completed. To get the data passing through IoT to cloudwatch we create a new policy and attach it to rule.
Make iot-logging-role.txt file and add,
```javascript
 {
  "Version":"2012-10-17",
  "Statement":[
	{
	  "Sid":"",
	  "Effect":"Allow",
	   "Principal":{
		"Service":"iot.amazonaws.com"
	    },
	    "Action":"sts:AssumeRole"
	  }
   ]
}

```
```javascript
aws iam create-role --role-name "iot-logging-role" --assume-role-policy-document file://iot-logging-role.txt
```
Adding logging policy to this rule,
Make file iot-logging-policy.txt and add,

```javascript
{
	"Version":"2012-10-17",
		"Statement":[
		   {
	"Effect":"Allow",
	"Action":[
	     "logs:CreateLogGroup",
	     "logs:CreateLogStream",
	     "logs:PutLogEvents",
	     "logs:PutMetricFilter",
	     "logs:PutRetentionPolicy",
	     "logs:GetLogEvents",
	     "logs:DeleteLogStream"
	],
	"Resource":[
		"*"
	]
     }
  ]
}
```
```javascript
 aws iam create-policy --policy-name "iot-logging-policy" --policy-document file://iot-logging-policy.txt
```
```javascript
 aws iam attach-role-policy --role-name "iot-logging-role" --policy-arn "arn:aws:iam::123456789012:policy/iot-logging-policy"
 ```
This logging option can be switched, on and off, for logging option on set ‘ logLevel=”INFO” ’.
For logging option off set ‘ logLevel=”DISABLED” ‘.

Test whether sensor readings are getting stored in DynamoDB run your python application.
An advantage of AWS IoT is also that we can send data to be stored on two applications simultaneously here I have tried to store sensor data on DynamoDB and to custom-endpoint at real time. You can describe your custom endpoint (optional).

Run
```javascript
 sudo python egrun.py
 ```
 On success, the data will be added in table (IOT-TempData) of database and can be viewed on DynamoDB console. Amazon CloudWatch logs will store the performance index of experiment.
 
 * To get a SMS notification on temperature reduce

Here we will create a rule, that will send sms to your mobile device when the temperature of room goes below 24 degrees Celsius.

Create SNS topic
```javascript
aws sns create-topic --name iot-hivedata-inside-temperature
 ```
 Create policy to publish this SNS topic and attach to iot-tempdata-actions-role created earlier.
Make file iot-hivedata-publish-sns-policy.txt and add,

```javascript
{
	"Version": "2012-10-17",
	"Statement": [ { "Effect": "Allow",
		       "Action": [ "sns:Publish" ],
		       "Resource": [ "arn:aws:sns:us-west-2:742485087494:iot-tempdata-temperature" ]
		}
	]
}

 ```
```javascript
aws iam create-policy --policy-name "iot-hivedata-publish-sns-policy" --policy-document file://iot-hivedata-publish-sns-policy.txt
 ```
```javascript
aws iam attach-role-policy --role-name "iot-hivedata-actions-role" --policy-arn "arn:aws:iam::123456789012:policy/iot-hivedata-publish-sns-policy"
 ```
 Create a rule that publishes to this topic when temperature goes below 24 degrees Celsius
Make file iot-tempdata-topic-sns-templessthan24-rule.txt and add,

 ```javascript
{
	"sql":"SELECT * FROM 'topic/tempdatatemperature' WHERE Temperature < 24",
	"ruleDisabled":false,
	"actions":[ {
		"sns":{
			"targetArn":"arn:aws:sns:us-west-2:123456789101:iot-tempdata-temperature",
			"roleArn":"arn:aws:iam:: 123456789101:role/iot-tempdata-actions-role"
		 }
		}
	]
}

 ```
```javascript
 aws iot create-topic-rule --rule-name "iot_tempdata_topic_sns_templessthan24_rule" --topic-rule-payload file://iot-tempdata-topic-sns-templessthan24-rule.txt
 ``` 
 Now we have all generated and attached policies and rules.
To publish our SNS topic to endpoint SMS enabled mobile device
For detailed steps, check  [here](http://docs.aws.amazon.com/iot/latest/developerguide/config-and-test-rules.html)

Go to AWS SNS console, you will notice iot-tempdata-temperature topic (created earlier) on console

Select this topic and from the Actions menu, choose Subscribe to topic

From Protocol choose SMS option and add your mobile number in this format 91-000-000-0000

Note: Here 91 is country code for India.

Click on ‘create subscription’

Go to AWS IoT console and select Create a rule, enter a name for your rule and simple description.

In Attribute field enter (*), which specifies to send entire MQTT message that triggered the rule.

In Topic filter, type iotbutton/your-button-DSN. If you are not using an AWS IoT button, type my/topic.

In condition field enter temperature < 24

From the Choose an action drop-down select Send message as a push notification (SNS)

From the SNS target select the Amazon SNS topic you created earlier

Choose the Create a new role link new webpage in IAM console will open, here accept the default values and select Allow,

Select Add action and then create.

Run
```javascript
sudo python egrun.py
 ``` 
On success, whenever the temperature received is less than 24 degree Celsius you will receive a SMS notification on described mobile number. While simultaneously all sensor reads are being stored in DynamoDB as usual.

Note: In egrun.py file the temperature, humidity and connected raspberry pi’s cpu serial number will be added to database and be sent in sms.


Mobile notification,

![Alt text](https://github.com/pdeolankar/Raspberry-pi-Temperature-sensor-AWS-IoT/blob/master/sms%20notification/img.PNG?raw=true ":")

![Alt text](https://github.com/pdeolankar/Raspberry-pi-Temperature-sensor-AWS-IoT/blob/master/sms%20notification/Screenshot_2016-12-20-14-43-58-207.jpeg?raw=true ":")


Experiment complete :+1:

## Activity 3

Send signal/message back from internet to raspberry pi to carry on a event.

## Activity 4

Transmit the IR code through IR transmitter with Raspberry Pi 2 to device compatible like A.C.
