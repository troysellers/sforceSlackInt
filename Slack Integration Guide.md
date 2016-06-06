#Salesforce Slack Integration

## Introduction

In this project we will be creating two different integrations between Salesforce and the Slack Messaging service. [Slack](http://www.slack.com) is a cloud based team collaboration tool, allowing teams to join and create different Channels. Slack is also highly extensible and provides some excellent mechanisms for integration, some of which we are going to explore in this project.  

To complete this project you will need 
* A <a href="https://developer.salesforce.com/signup" target="_blank">Salesforce<a/> Developer Edition account
* An <a href="https://heroku.com" target="_blank">Heroku</a> account
* A <a href="https://slack.com" target="_blank">Slack</a> account 

There are three integrations that you are going to build:
* Salesforce to Slack - notify on update of an Opportunity stage.
* Slack Slash Commands - provide a set of commands that Slack users can execute to see Salesforce data in Slack
* Configure a Salesforce bot to monitor Slack channels and respond

Sounds like a lot? Well, luckily some of the <a href="http://coenraets.org/blog/2016/04/salesforce-slack-bot/" target="_blank">heavy lifting</a> has been done for us and today we will be able to stand on the shoulders of giants as we go.

Let's get started! 

## 1 - Notify Slack Team of Opportunity Status Update
Picture this, your project team is happily working away delivering for their customers while your sales team is on the road, closing deals. The project team have a very active Slack channel that they use to keep on top of all the changing requirements for in flight projects, as well as helping them manage resourcing when changes happen. The team is always interested when deals move towards close, it helps them keep one eye on the resourcing pipeline. Lets build a simple integration to make it even easier for the project team to stay up to date.

### What you will do
1. Create a Slack Team & Channel
2. Add the Slack Webhook 
3. Create an Apex Class
4. Build the Process
5. Enable Remote Site and Test

![Salesforce to Slack](5.6 - salesforceToSlack.png)

## The Slack Bit...
### Create Slack Team and Channel
<a href="https://slack.com/create" target="_blank">Create</a> your own Slack account for this exercise, your company might already have teams on Slack so when you register with your email address you could be prompted to join one of these existing channels. They may not appreciate your positing updates from your Salesforce Developer environment so lets create a new team from scratch. Go ahead and let the wizard guide you through creation of your own Slack Team, you should end up with details similar to this. 

![Create Slack Team](3 - Create Team.png)

Oh, and if you're the type that likes to clean up after yourself we can <a href="https://my.slack.com/admin/settings" target="_blank">delete</a> our Slack team as well..... please don't be deleting any working, production Slack teams will you.

Once you have created your team, you should see that you have already been added to some default Channels. Slack Channels are an effective way of broadcasting information to a group of users who are interested, they express their interest by subscribing to channels they are interested. 

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
Now we have an endpoint configured and listening for something, its time to configure Salesforce to send the messages to Slack. We are going to do this by writing a small piece of Apex code that will be fired from a process we define in the Process Builder. Before we get started, if you haven't got a developer edition of Salesforce by now (what have you been doing all day?) then jump over to <a href="https://developer.salesforce.com/signup" target="_blank">signup</a> and get a new one.

###Apex Class
Now we can create the Apex code that is capable of posting a message to this newly configured Webhook URL. The methods and classes here will allow Opportunity Name and Stage fields to be posted out to the the URL. Copy and paste this code if you don't feel like typing it out :)

You will need to take the Webhook URL from the step above and add that into this class.

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
Notice the <a href="https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_annotation_InvocableVariable.htm" target="_blank">@InvocableVariable</a> and <a href="https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_classes_annotation_InvocableMethod.htm" target="_blank">@InvocableMethod</a> annotations on this class that allow these methods to be exposed to the configuration tools in the Salesforce system. Other interesting Apex features in this code are the use of <a href="https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_class_System_Queueable.htm" target="_blank">System.Queueable</a> and <a href="https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_batch_interface.htm" target="_blank">Database.AllowsCallouts</a> interfaces that are being used. 

Oh, remember that we are are all about security here at Salesforce as well, so lets go ahead and explicity allow this callout to happen by setting the Remote Site as well.  (Setup -> Security Controls -> Remote Site Settings then click New)

![Create Process](5.01 - remote site config.png)


### Build the Process
Now we can define the business process that will cause the notifcation to fire and be propogated into Slack. Here we get to see one of the really powerful features of Salesforce when you writing code, the amount of code you don't have to write! As a developer all we needed to do was build a small module that took some parameters and passed them to our Slack endpoint, the who, when and why of this integration is now completely declarative! 

Lets fire up our Process Builder and create this rule.
![Create Process](5.0 - create process.png)

Click New and populate the details of your new Process.

![New Process](5.2 - new process window.png)

Select the Opportunity object, and choose to fire whenever a record is created or updated, then add the selection criteria. i.e. When does the action need to fire? In our case, we want to notify our project teams through Slack when an Opportunity record changes stage. Notice here we could add plenty of different conditions if required, we might not want to spam our team with every minor detail? 

![Process Criteria](5.3 - Process Criteria.png)

Now we can add an action that calls our fresly minted Apex class, ready to accept the two parameters that we annotated with the @InnvocableVariable annotation.
![Set the Action](5.4 - Configure Apex Class.png)

Once this is complete, remember to *Activate your process* so we can test that this is all working.

### Test
Your functioning integration should now be ready to test. Go ahead and login to Slack

<a href="https://youtu.be/M8gEkDk0bto" target="_blank">![Process Builder](6.3 - Step1Video.png)</a>


## 2 - View Salesforce Data Using Slash Commands
Now our project team is fully aware of the latest and greatest news on deals in real time from Salesforce, but what if they wanted to interact with Salesforce data themselves? Should these users really have to leave their beloved Slack interface if they just wanted to see a few opportunities, contacts or to create a simple case? Luckily we have a few handy developers on staff who can pull together a little integration that will allow just this. Lets have a look at <a href="https://api.slack.com/slash-commands" target="_blank">Slack Slash Commands</a> and another awesome component from the Salesforce App Cloud, Heroku. 

We want to setup a few different scenarios here :
* Show the top opportunities from Salesforce  (/pipeline[number to show])
* Search for a Salesforce Contact in the Slack UI (/contact[search key])
* Create a customer service Case from the Slack UI (/case[subject:description])

To achieve this we are going to :
* Setup a Salesforce Connected App
* Create a Node.js application that will serve as the proxy between Salesforce and Slack (well, actually we are just going to copy one!)
* Configure Slash Commands in Slack

![Slack to Salesforce](6.4 - Slack to Salesforce.png)

### Architecture
We need to setup a small Heroku app to broker the communcation between Slack and Salesforce. This app is going to use <a href="https://nodejs.org/" target="_blank">Node.js</a> and the <a href="https://github.com/kevinohara80/nforce" target="_blank">nForce</a> module to provide convenience methods for accessing Salesforce. If you have yet to get your <a href="https://heroku.com" target="_blank">Heroku</a> account, head over and sign up for the free tier. 

![Heroku Sign Up](7.1 - sIgnupHeroku.png)

We need to let Salesforce know that an application is going to want to use the API, so we have to configure a the Connected App in our developer environment. Go to Apps to create a new Connected App. 

![New Connected App](7.2 - ConnectAppConfig.png)

Configure, don't worry about the callback URL yet as we are going to change that later! 

![Configure Connected App](7.3 - ConnectedAppConfig.png)

Now we can deploy this <a href="https://github.com/ccoenraets/slackforce" target="_blank">application</a> into our newly (or well used) Heroku environment, it is a trivial thing to deploy this appliction thanks to Heroku... try it, just click this button. 

<a href="https://heroku.com/deploy?template=https://github.com/ccoenraets/slackforce" target="_blank">![Deploy](https://www.herokucdn.com/deploy/button.svg)</a>

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

Click install and then add the following commands to your Team. (remember, your app name will be an individual little snowflake!)


Command | URL | Method | Custom Name
--------|-----|--------|------------
/pipeline | https://app_name.herokuapp.com/pipeline | POST | Top Opportunities
/contact | https://app_name.herokuapp.com/contact | POST | Salesforce Contacts
/case | https://app_name.herokuapp.com/case | POST | Salesforce Cases

Once you create your first Slash command you will land on the configuration page, this has all the details you are going to need for configuration of the Heroku app. To create a second (or third!) navigate back to the "Cofigurations On [your slack team]" section using the breadcrumb navigation, then add another configuration.

You will need to create a separate Slash Command for each use case, configuring a Slash command should look like this. This is also where you will get you Slack Token that needs to be configured in your Heroku application.

![Configure](8.2 - slashConfig.png)

Now we have created our Slash Commands, lets update our Heroku app with the Tokens that were just generated for each command. From the Settings tab in your <a href="dashboard.heroku.com" target="_blank">Heroku Dashboard</a>, reveal the config vars and edit. Oh, and don't forget to generate and append your <a href="https://developer.salesforce.com/docs/atlas.en-us.api.meta/api/sforce_api_concepts_security.htm" target="_blank">security token</a> to the SF_PASSWORD variable!!

![Tokens](8.3 - TokensInHeroku.png)

Lastly, remember we said there was a Connected App setting we needed to come back to? Thats right, our callback URL is not a localhost address anymore but should now point to our Heroku app.

![Callback](8.4 - CallbackScope.png)

Congratulations, you should now be able to pull Salesforce data into your Slack stream.

[![Salesforce in Slack](8.5 - Step2Video.png)](https://youtu.be/xB-1SsUoBHk)


## 3 - Its all in the Bot

In the last part of our workshop we are going to create an integration using bots, we are going to monitor Slack channels and respond to Salesforce requests expressed in natural language. 

We are going to create an applciation that opens a Websocket connection to Slack, allowing the bot to listen to the Slack channel as well as any direct messages that users send directly to the bot. We are going to use <a href="https://github.com/howdyai/botkit" target="_blank">Botkit</a> to abstract some of the low level details of Slack (or <a href="https://blog.howdy.ai/botkit-for-facebook-messenger-17627853786b" target="_blank">Facebook</a>) bot buildling. 

To finish off today, we are going to :
* Configure another Salesforce Connected App
* Create another Heroku App
* Create a bot user in Slack

### 1 - Create a Salesforce Connected App

Create Connected app.

![Connected App Config](9.1 - ConnectedApp Config.png)


### 2 - Create the Slack Bot User

Lets add a bot to our Slack Team. 

![Add the Bot](9.2 - AddTheBot.png)

### 3 - Create the Heroku App

Go ahead and use your Heroku again (or follow this for deploying locally)

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https://github.com/ibigfoot/salesforce-bot-slack)

Setup your config vars, again to remember that your security token should be appended to the password.

![Heroku Config](9.3 - HerokuAppConfig.png)


You should now be able to have a conversation with your Salesforce bot!

[![Salesforce Bot](10 - Step3Video.png)](https://youtu.be/1EPNbHi-3UY)


And that's it! Well done if you have made it this far. 

