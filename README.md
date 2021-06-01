# ServerlessDays NYC
Below are the instructions for the IBM Cloud Functions & Apache OpenWhisk lab at Serverless Days NYC 2018.

Feel free to reach out with any questions!


### Sign up for IBM Cloud Account
*IBM Cloud Functions is based on the Apache OpenWhisk project.  We'll be using IBM Cloud Functions for today's lab, but the concepts could apply to any managed Serverless offering.*

1. [Sign up for an IBM Cloud Account](https://ibm.biz/BdY3BT)
2. This should redirect you to IBM Cloud, but if not you can go directly there at [https://bluemix.net](https://bluemix.net)
3. Sign in to confirm your account was created.

### Install IBM Cloud CLI & IBM Cloud Functions Plugin
*In this section, you will install the IBM Cloud CLI and the IBM Cloud Functions Plugin to the CLI, as well as do some account set up and configuration. This will enable you to interact with IBM Cloud Functions from a command line interface.*

1. Install the IBM Cloud CLI 
    1. From Shell:
	    * Mac/Linux: In your terminal window, run `curl -sL https://ibm.biz/idt-installer | bash`
	    * Windows 10 Pro, run `Set-ExecutionPolicy Unrestricted; iex(New-Object Net.WebClient).DownloadString('http://ibm.biz/idt-win-installer')`

    2. Alternatively, you can use an installer:
        * Link to installer: [Link to installer](https://console.bluemix.net/docs/cli/reference/ibmcloud/download_cli.html#install_use)

2. Configure your environment
    * Select your API: `ibmcloud api https://api.ng.bluemix.net`
    * Login: `ibmcloud login`
    * You should have an org created already. Target your org with the command `ibmcloud target --cf`
    * You should have a space created as well, called dev.  If you did, then the `ibmcloud target --cf` command would show that this space was targeted, and you can skip the steps for creating a space.
      * If you do not have a space; let's create one. In this example, we'll call our space dev, but you can choose anything you want: `ibmcloud cf create-space dev`
      * Ensure your org & space is correctly targeted using `ibmcloud target --cf`
      * Confirm cloud-functions plugin is installed: `ibmcloud fn` should return some help information.
          * If the plugin is not installed, install it: `ibmcloud plugin install cloud-functions`

### Create your first action (Hello World) using the CLI!
*Your first goal is to create a simple hello world action.  This action will return the string 'Hello World' when it is in invoked.*

1. Create a folder for your action to live in: `mkdir myFolder && cd myFolder`
2. Create a file called helloworld.js: `touch helloworld.js`
3. Use your favorite editor to paste the following code into helloworld.js:

	```
	function main(params) {  
		if (params.name) {    
			return { greeting: `Hello ${params.name}` };  
		}  
			return { greeting: 'Hello stranger!' };
		}
	```
4. This code will return "Hello stranger" if no name parameter is given, otherwise it will say hello to the supplied name. Let's create a new action using this file: `ibmcloud fn action create helloWorld helloworld.js`
5. Let's invoke the action hosted on IBM Cloud Functions, using -r to wait for a result: `ibmcloud fn action invoke helloWorld -r`
6. You should see a response like:

	```
	{
	    "greeting": "Hello stranger!"
	}
	```
7. Let's invoke the action with a parameter of your name: `ibmcloud fn action invoke helloWorld -p name Belinda -r`
8. You should see a response like:

	```
	{
	    "greeting": "Hello Belinda"
	}
	```

### Create an action that fires in response to a Kafka trigger
*You can imagine that you could do something a little more interesting than hello world.  For this section of the lab, you'll be creating a trigger that will fire whenever a new item is placed on a Kafka topic.  That trigger will be connected to an action (via a rule).  The action will run some code to process the incoming kafka message.*

1. Create a folder for this project, and create a file named kafkaAction.js: `mkdir myFolder && cd myFolder && touch kafkaAction.js`
2. Open the kafkaAction.js file in your favorite editor & paste the following code into the file

	```
	function main(params) {
	  if (params.messages) {
	    return { message: params.messages[0].value,
		     topic: params.messages[0].topic };
	  }
	}
	```
3. Save the file.  
4. Create an action using the file: `ibmcloud fn action create kafkaAction kafkaAction.js`
5. Create trigger to listen to messages coming in on the kafka topic
  * To create the trigger, you will first need a package binding, which will store the credentials for the kafka instance.  For simplicity in this lab, we are using a kafka instance already created by the instructors. Copy paste the following command into the terminal to create the package binding:

	  ```
	  ibmcloud fn package bind /whisk.system/messaging myMessageHub -p kafka_brokers_sasl "[\"kafka03-prod02.messagehub.services.us-south.bluemix.net:9093\", \"kafka02-prod02.messagehub.services.us-south.bluemix.net:9093\", \"kafka04-prod02.messagehub.services.us-south.bluemix.net:9093\", \"kafka01-prod02.messagehub.services.us-south.bluemix.net:9093\", \"kafka05-prod02.messagehub.services.us-south.bluemix.net:9093\"]" -p user HFrcTJyrKd0GRNkK -p password 5NJkATVxktFCnN79Gxa1flhe9GDWJE8s -p kafka_admin_url "https://kafka-admin-prod02.messagehub.services.us-south.bluemix.net:443"
	  ```
  * Create the trigger using the package binding you just created to listen to the topic named "mytopic"

	  ```
	  ibmcloud fn trigger create myMessageHubTrigger -f myMessageHub/messageHubFeed -p topic mytopic -p isJSONData true
	  ```

6. The trigger needs to be connected to the action via a rule, so that the action will be run whenever a new item is on the kafka topic:

	```
	ibmcloud fn rule create myRule myMessageHubTrigger kafkaAction
	```

7. The instructors will be sending messages to the kafka topic periodically.  You can see your action firing a couple of different ways:
	* Look at the activations:
	  * `ibmcloud fn activation list` to see something like this:


			activations
			02e1e4fa56dc49c1a1e4fa56dcc9c1d2 kafkaAction          
			9c10e5ac5c21449690e5ac5c21c49611 myMessageHubTrigger


	As you can see, the trigger was fired, which cased the kafkaAction to be fired due to the rule we created.
	  * See the output for the kafkaAction activation: `ibmcloud fn activation get  02e1e4fa56dc49c1a1e4fa56dcc9c1d2` to see something like this:


		  	"response": {
		  		"status": "success",
		  		"statusCode": 0,
		  		"success": true,
		  		"result": {
					"message": "\"This is the content of my message\"",
					"topic": "mytopic"
				}
        		}


		  As you can see, the response contains our expected message and topic.

	* You can also go view the [monitor dashboard](https://console.bluemix.net/openwhisk/dashboard) to see your recent invocations.
