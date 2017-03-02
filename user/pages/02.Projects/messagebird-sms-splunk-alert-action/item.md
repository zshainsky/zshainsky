---
title: 'MessageBird SMS + Splunk  Custom Alert Action'
---

### MessageBird SMS API
MessageBird is a software comany that enables anyone from a developer to en entrire enterprise to communicate with customers through simple to use APIs for SMS, chat, voice, and interactive bots. MessageBird's SMS API allows developers to send SMS messages from an "originator" (the person sending the message) to  a list of "recipients" (the person or people recieving the message) in a simple to use API. For this project I will be using their Python REST API to send messages. I chose python because it plays very nicely with Splunk...and I am familiar with it :).

#### Getting Started: SMS API Python Example:
The code below was altered from the [MessageBird SMS API Documentation](https://developers.messagebird.com/docs/messaging#messaging-send) on how to send a message with Python. A few things to remember, the 'originator' value is either a telephone number or an alphanumeric string with a max length of 11. Also, if you are sending to numbers with in the USA and a few other countries you must request access, but this is all documented when you sign up and go through the "setup guide" on MessageBirds admin portal. Lastly, before you are able to run your code, you need to download the [MessageBird Python API](https://github.com/messagebird/python-rest-api). Follow the install instructions here. I used `pip install messagebird`. Make sure you know which site-packages folder `pip` will downloads to. If you dont run this script with the correct python that references the site-packages folder pip downloads to, python wont be able to find the messagebird package. This also becomes important with including libraries in Splunk's python path, but we will get to that in a minute.

```python
import messagebird

ACCESS_KEY = 'Insert your Access Key here'
ORIGINATOR = '15031234567'
BODY = 'This is a test message.'
RECIPIENTS = '15031234567'

client = messagebird.Client(ACCESS_KEY)
message = client.message_create(
        ORIGINATOR,
        RECIPIENTS,
        BODY,
        { 'reference' : 'Foobar' }
    )
```
[MessageBird API Documentation](https://developers.messagebird.com/docs/)

Testing the script in terminal. For me `/usr/local/bin/python3.4` was the correct python that could find where pip installed the MessageBird API:
```
$ /usr/local/bin/python3.4 /path/to/above/script.py
```

### Splunk Custom Alert Actions
Splunk is a universal machine data platform that is used to collect, analyize, report and alert on events in any type of environment. Alerts in Splunk are extremely flexible as they are powered by any type of Splunk search, using the Splunk Processing Language (SPL). When an alert is triggered in Splunk, there is a specific action that is specified by the user as far as what Splunk should do upon triggering. There are a number of prebuilt options for Splunk alerts such as, send an email, run a script, and alert in the Splunk console. These are all fun and useful but Splunk also offeres the ability to create CUSTOM Alert Actions. So, being a curious developer that is certainly a more interesting avanue to explore than pre built Splunky Goodness.

#### Getting Started: Custom Alert Action
The best place to start with Custom Alert Actions in Splunk is the [documentation](http://docs.splunk.com/Documentation/Splunk/6.5.1/AdvancedDev/ModAlertsIntro). For real...read it, otherwise you are up the creek without a paddle. Once you have read it the best thing you can do next is look at the [exmples](http://docs.splunk.com/Documentation/Splunk/6.5.1/AdvancedDev/ModAlertsBasicExample). I was fortunate to find another Custom Alert Action that is designed to send SMS for Twilio on [Splunkbase.com](http://splunkbase.com), which was an excelent guide.

The first place that I stared with creating a Custom Alert Action that functioned. Essentially, I copied the Logger example to get started and began to build on top of that once I had it working. I would definitely start with the logging example or at least implement the log method into your code because visibility into how far your python code gets becomes very slim once Splunk takes charge of running your script.
```
def log(msg):
    f = open(os.path.join(os.environ["SPLUNK_HOME"], "var", "log", "splunk", "messagebird_sms.log"), "a")
    print >> f, str(datetime.datetime.now().isoformat()), msg
    f.close()
```
There are two ways to get visibility into what happens when your script runs:
1. Using the above logging function. This is a nice solution because Splunk is already monitoring all of the files in that specific directory `$SPLUNK_HOME/var/log/splunk/`. You can see this if you navigate to the _"Settings > Data Inputs > Files & directories"_ in the Splunk UI. Because Splunk is already monitoring this directory, you can start searching for the events that are getting logged (essentiall any output you want to see from your script) just from a simple Splunk search: ``` index=_internal source=*messagebird_sms.log```.
2. The other way, and perhaps most recomended way but less comfortable for me, is to print your output to the STDERR of your system. This was less comfortable to me because the only way I could find the results from that was through the Splunk job inspector and if you are unfamiliar with the job inspector, it might not be the most friendly way to debug your script. This requires you to run the Splunk command (which I will explain further below) `| sendalert <alert name> [options]`. After you have done this you can select the job inspector, "Job > Inspect Job", where you will be greated by a pop up window. Scroll to the bottom and expand the "Search job properties". Then scroll to the bottom of that and select "search.log". Here you can find the output that you wrote in your script to the STDERR. Phew, that was a lot...if you didn't catch that...dont worry and follow the first option.

#### Testing your Custom Alert Action in Splunk
##### Install Custom Alert
Once you have set up your Custom Alert Action, you can install it the same way you would install any other Splunk app. Either tar.gz the folder, or simply copy the folder into `$SPLUNK_HOME/etc/apps/`.

##### Allowing Splunk to import the MessageBird Python API
You have already tested the MessageBird Python API at the top of this page to confirm that we can indeed create a connection with our Access Key and get some response back after creating a message, but this was done outside of Splunk. Splunk has it's own python interpreter that references its own _site-packages_ folder. This folder is what python uses to look for libraries when the `import` command is used. If there is no _messagebird_ folder inside the proper _site-packages_, your script will error out at that line and you will not be a happy camper.

The trick here is to download the zip file from MessageBird's github [here](https://github.com/messagebird/python-rest-api) (this is the alternate way of installing packages in Python without using `pip`) and unzip the file. Then move the _messagebird_ folder into your Alert Action's _bin_ folder. You can find this at `$SPLUNK_HOME/etc/<your alert action folder name>/bin/`. This is also where your Alert Action script will be located.

##### Testing...
In order to test you could either:
1. Set up an alert to run on some schedule or in real time. This is a fine solution but doesn't give you fine tuned control over when you want the alert to trigger. 
2. Run this command `| sendalert <alert name> [options]` [sendalert Documentation](https://docs.splunk.com/Documentation/Splunk/6.5.1/SearchReference/Sendalert). Here is an example of my test command
```
index=_internal | sendalert messagebird_sms param.body="Hello world. Can you see my body?" param.originator="15031234567" param.recipients="15031234567" param.accesskey="<your access key here>"
```
The _param.<name>_ options give you the ability to simulate the fields specified in the html template that users input information into. In the messagebird Alert Action I am expeciting a user to input a body, originator, recipient and accesskey. The accesskey gets stored on setup through the _setup.xml_ file.

####  Result
At this point I am still waiting to get access to send messages with in the USA, but here is the functioning code (leaving in log output to help explain whats going on):
```
import messagebird
import sys, os, datetime
import json

'''
send_message function is designed to establish the connection to the messagebird client and create the api call to send an SMS message
Param (configuration): JSON object
Return: Returns True or False, whether message was sent properly or not
'''
def send_message(configuration):
    log("Sending message...")
    # Get all variables from the configuration variable which is a JSON object
    ACCESS_KEY = configuration.get('accesskey')
    log(ACCESS_KEY)
    ORIGINATOR = configuration.get('originator')
    log(ORIGINATOR)
    RECIPIENTS = configuration.get('recipients')
    log(RECIPIENTS)
    BODY = configuration.get('body')
    log("(Access Key: %s, Originator: %s, Recipients: %s, Body: %s)" % (ACCESS_KEY, ORIGINATOR, RECIPIENTS, BODY))

    # Test connection to messagebird client and create message with above criteria
    try:
        client = messagebird.Client(ACCESS_KEY)
        message = client.message_create(ORIGINATOR, RECIPIENTS, BODY, { 'reference' : 'Foobar' })
        log(message)
    except messagebird.client.ErrorException as e:
        # Trouble in Rivercity
        log('\nAn error occured while requesting a Message object:\n')
        for error in e.errors:
            log('  code        : %d' % error.code)
            log('  description : %s' % error.description)
            log('  parameter   : %s\n' % error.parameter)
    # Extra logic will be added for the return value
    return True

'''
log function writes data to $SPLUNK_HOME/var/log/splunk/messagebird_sms.log file
Param (msg): String to write to file
'''
def log(msg):
    f = open(os.path.join(os.environ["SPLUNK_HOME"], "var", "log", "splunk", "messagebird_sms.log"), "a")
    print >> f, str(datetime.datetime.now().isoformat()), msg
    f.close()


if __name__ == "__main__":
    log("Starting MessageBird SMS Alert Action...")
    log("Got arguments: %s" % sys.argv)
    payload = sys.stdin.read()
    log("Got payload: %s" % payload)

    # Read commandline args to make sure --execute mode is established. If not, dont run.
    if len(sys.argv) > 1 and sys.argv[1] == "--execute":
        try:
            # Convert JSON payload from stdin to a JSON Object
            payload = json.loads(payload)
        except ValueError as e:
            log("JSON Loads Error: %s" % e)

        # print >> sys.stderr, "INFO Hello STDERR"
        log("Past JSON Loads...")
        # Grab configuration object, which contains all params sent by Splunk (ie: accesskey, originator, recipients, body)
        configuration = payload.get('configuration')
        log(configuration)
        # Send message command
        if not send_message(configuration):
            log("Issue sending message")
            sys.exit(2)
        else:
            log("Message sent successfully")
    else:
        log("Unsupported execution mode (expected --execute flag)")
        sys.exit(1)
```
