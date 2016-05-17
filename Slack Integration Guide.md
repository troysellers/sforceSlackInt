#Salesforce Slack Integration


## Introduction

In this project we will be creating two different integrations between Salesforce and the Slack Messaging service. [Slack](http://www.slack.com) is a cloud based team collaboration tool, allowing teams to join and create different Channels. Slack is also highly extensible and provides some excellent mechanisms for integration, some of which we are going to explore in this project.

There are three integrations that you are going to build:
* Salesforce to Slack - notify on update of an Opportunity stage.
* Slack Slash Commands - provide a set of commands that Slack users can execute to see Salesforce data in Slack
* Configure a Salesforce Bot to monitor Slack channels and respond

Sounds like a lot? Well, luckily some of the [heavy lifting](http://coenraets.org/blog/2016/04/salesforce-slack-bot/) has been done for us and today we will be able to stand on the shoulders of giants as we go.

Let's get started! 



## 1 - Notify Slack Team of Opportunity Status Update
### What you will do
1. Create a Slack Team & Channel
2. Add the Slack Webhook 
3. Create an Apex Class
4. Build the Process
5. Enable Remote Site and Test

## The Slack Bit...
### Create Slack Team and Channel
Create your own Slack account for this exercise, your company might already have teams on Slack so when you register with your email address you could be prompted to join one of these existing channels. Go ahead and let the wizard guide you through creation of your own Slack Team, you should end up with something like this.

![Create Slack Team](3 - Create Team.png)

Once you have created your team, go ahead and add a new Channel. This will allow users to subscribe to content being published from your Salesforce integration.

![Create Slack Channel](1 - Create Slack Channel.png)

### Create Slack Integration
Once you have your team and channel in place you will need to add an integration, this is an endpoint for Salesforce to call when it has something interesting to report. Click on the gear icon in the top right hand corner and select "Add an app or integration".

![Add the Integration](4.1 - Add an integration.png)

Select the integration you need from the list provided, we want to add the "Incoming Webhooks" app to our Team. Click "Install" to add this to your environment.

![Install Webhooks](4.2 - Install Webhook.png)

Lastly, we need to configure an endpoint for Salesforce to call. The configuration allows you to select a Channel to POST into, set some labels and also provides the endpoint URL that you will need later in the tutorial.

![Configure Webhook](4.3 - Webhook settings.png)

Now you should have a new Slack Team, Channel and a configured webhook that is ready to go. Before we get to the fun stuff (i.e. Salesforce!) we can test our Webhook is working correctly with a simple cURL command from the commandline. 

```
curl -X POST -H 'Content-type: application/json' --data '{"text":"This is a test post from cURL"}' https://hooks.slack.com/services/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
```
Should result in something like this.

![Testing With cURL](4.4 - cURL test.png)

## The Salesforce Bit...
Now we have an endpoint configured and listening for something, its time to configure Salesforce to send the messages to Slack. We are going to do this by writing a small piece of Apex code that will be fired from a process we define in the Process Builder

###Apex Class
Now we can create the Apex code that is capable of posting a message to this newly configured Webhook URL. The methods and classes here will allow Opportunity Name and Stage fields to be posted out to the the URL. 

```
public with sharing class SlackOpportunityPublisher {
     
    private static final String slackURL = 'YOUR_WEBHOOK_URL';
     
    public class Oppty {
        @InvocableVariable(label='Opportunity Name')
        public String opptyName;
        @InvocableVariable(label='Stage')
        public String stage;
    }
     
    @InvocableMethod(label='Post to Slack')
    public static void postToSlack(List<Oppty> oppties) {
        Oppty o = oppties[0]; // If bulk, only post first to avoid overloading Slack channel
        Map<String,Object> msg = new Map<String,Object>();
        msg.put('text', 'The following opportunity has changed:\n' + o.opptyName + '\nNew Stage: *' + o.stage + '*');
        msg.put('mrkdwn', true);
        String body = JSON.serialize(msg);    
        System.enqueueJob(new QueueableSlackCall(slackURL, 'POST', body));
    }
     
    public class QueueableSlackCall implements System.Queueable, Database.AllowsCallouts {
         
        private final String url;
        private final String method;
        private final String body;
         
        public QueueableSlackCall(String url, String method, String body) {
            this.url = url;
            this.method = method;
            this.body = body;
        }
         
        public void execute(System.QueueableContext ctx) {
            HttpRequest req = new HttpRequest();
            req.setEndpoint(url);
            req.setMethod(method);
            req.setBody(body);
            Http http = new Http();
            HttpResponse res = http.send(req);
        }
 
    }
    
}
```
Notice the [@InvocableVariable](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_annotation_InvocableVariable.htm) and [@InvocableMethod](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_annotation_InvocableMethod.htm) annotations on this class that allow these methods to be exposed to the configuration tools in the Salesforce system. Other interesting Apex features in this code are the use of [System.Queueable](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_class_System_Queueable.htm%23apex_class_System_Queueable) and [Database.AllowsCallouts](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_batch_interface.htm) interfaces that are being used. 


### Build the Process
Now we can define the business process that will cause the notifcation to fire and be propogated into Slack. 

Lets fire up our Process Builder and create this rule.
![Create Process](5.0 - create process.png)

Click New and populate the details of your new Process.
![New Process](5.1 - new process.png)

![New Process](5.2 - new process window.png)

Select the Opportunity object, then add the selection criteria. i.e. When does the action need to fire? In our case, we want to set when the opportunity stage has changed.

![Process Criteria](5.3 - Process Criteria.png)

Now lets use that Apex class we created, populate the variables and execute the call to Slack!
![Set the Action](5.4 - Configure Apex Class.png)

### Test
Your functioning integration should now be ready to test. 

[![Process Builder](6.3 - Step1Video.png)](https://youtu.be/M8gEkDk0bto)


## 2 - View Salesforce Data Using Slash Commands
So we have a conversation going from Salesforce to Slack, but what about a user that wants to pull data into Slack without leaving Slack? Well, lets have a look at [Slack Slash Commands](https://api.slack.com/slash-commands).
We want to setup a few different scenarios here :
* Show the top opportunities from Salesforce  (/pipeline[number to show])
* Search for a Salesforce Contact in the Slack UI (/contact[search key])
* Create a customer service Case from the Slack UI (/case[subject:description])

To achieve this we are going to :
* Setup a Salesforce Connected App
* Create a Node.js application that will serve as the proxy between Salesforce and Slack (well, actually we are just going to copy one!)
* Configure Slash Commands in Slack

### Architecture
We need to setup a small Heroku app to broker the communcation between Slack and Salesforce. This app is going to use [Node.js](https://nodejs.org/) and the [nForce](https://github.com/kevinohara80/nforce) module to provide convenience methods for accessing Salesforce. If you have yet to get your [Heroku](www.heroku.com) account, head over and sign up for the free tier. 

![Heroku Sign Up](7.1 - sIgnupHeroku.png)

We need to let Salesforce know that an application is going to want to use the API, so we have to configure a the Connected App in our developer environment. Go to Apps to create a new Connected App.

![New Connected App](7.2 - ConnectAppConfig.png)

Configure, don't worry about the callback URL yet as we are going to change that later! 

![Configure Connected App](7.3 - ConnectedAppConfig.png)

Now we can deploy the [application](https://github.com/ccoenraets/slackforce), it is a trivial thing to deploy this appliction thanks to Heroku... try it, just click this button. 

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https://github.com/ccoenraets/slackforce)

You will be asked to configure some properites on this application

* SF_CLIENT_ID - enter the Consumer Key of your Salesforce Connected App
* SF_CLIENT_SECRET - enter the Consumer Secret of you Salesforce Connected App
* SF_USER_NAME - this is the username of the Salesforce integration user
* SF_PASSWORD - this is the password for this user
* SLACK_OPPORTUNITY_TOKEN, SLACK_CONTACT_TOKEN and SLACK_CASE_TOKEN are blank for now (we get back to this)

Once you have deployed, you should see the an output similar to this. 

![Success Heroku](7.4 - SuccessDeploy.png)

### Create the Slack Commands
We now need to do some configuration work in the Slack UI to create the actual Slack commands that end users will need. In your browser, open up Slack (if you haven't already). As an example, my Slack team URL is https://sforce-slack-demo.slack.com. We are going to add another integration, however this time we will add Slash Command (remember our first one was a Webhook)

![Add Integration](8.0 - AddIntegration.png)

![Add Slash Command](8.1 - SlashCommand.png)

Click install and then add the following commands to your Team.


Command | URL | Method | Custom Name
--------|-----|--------|------------
/pipeline | https://app_name.herokuapp.com/pipeline | POST | Top Opportunities
/contact | https://app_name.herokuapp.com/contact | POST | Salesforce Contacts
/case | https://app_name.herokuapp.com/case | POST | Salesforce Cases


![Configure](8.2 - slashConfig.png)

Now we have created our Slash Commands, lets update our Heroku app with the Tokens that were just generated for each command. From the Settings tab in your [Heroku Dashboard](dashboard.heroku.com), reveal the config vars and edit. Oh, and don't forget to generate and append your security token to the SF_PASSWORD variable!!

![Tokens](8.3 - TokensInHeroku.png)

Lastly, remember we said there was a Connected App setting we needed to come back to? Thats right, our callback URL is not a localhost address anymore but should now point to our Heroku app.

![Callback](8.4 - CallbackScope.png)

Congratulations, you should now be able to pull Salesforce data into your Slack stream.

[![Salesforce in Slack](8.5 - Step2Video.png)](https://youtu.be/xB-1SsUoBHk)


## Something Else 