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
[![IMAGE ALT TEXT HERE](http://img.youtube.com/vi/YOUTUBE_VIDEO_ID_HERE/0.jpg)](http://www.youtube.com/watch?v=YOUTUBE_VIDEO_ID_HERE)

## View Salesforce Data Using Slash Commands

### Register For Heroku 

### Get Your Heroku App

### Create Your Slash Commands

### Test

## Something Else 