#Salesforce Slack Integration


## Introduction

In this project we will be creating two different integrations between Salesforce and the Slack Messaging service. [Slack](http://www.slack.com) is a cloud based team collaboration tool, allowing teams to join and create different Channels. Slack is also highly extensible and provides some excellent mechanisms for integration, some of which we are going to explore in this project.

There are three integrations that you are going to build:
* Salesforce to Slack - notify on update of an Opportunity stage.
* Slack Slash Commands - 



## Notify Slack Team of Opportunity Status Update
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

### Build Process

### Test


## View Salesforce Data Using Slash Commands

### Register For Heroku 

### Get Your Heroku App

### Create Your Slash Commands

### Test

## Something Else 