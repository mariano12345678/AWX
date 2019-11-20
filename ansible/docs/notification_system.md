# Notification System Overview

A Notification Template is an instance of a notification type (Email, Slack, Webhook, etc.) with a name, description, and a defined configuration. A few examples include:

* Username, password, server, recipients for the Email type.
* Token and list of channels for Slack.
* URL and Headers for webhooks.

At a high level, the typical notification task flow is:

* User creates a `NotificationTemplate` at `/api/v2/notification_templates/`.
* User assigns the notification to any of the various objects that support it (all variants of Job Templates as well as organizations and projects) and at the appropriate trigger level for which they want the notification (error, success, or any). For example, a user may wish to assign a particular Notification Template to trigger when `Job Template 1` fails.

## Notification Hierarchy

Notification templates assigned at certain levels will inherit notifications defined on parent objects as such:

* Job Templates will use notifications defined on it as well as inheriting notifications from the Project used by the Job Template and from the Organization that it is listed under (via the Project).
* Project Updates will use notifications defined on the project and will inherit notifications from the Organization associated with it.
* Inventory Updates will use notifications defined on the Organization it is in.
* Ad-hoc commands will use notifications defined on the Organization with which that inventory is associated.

## Workflow

When a job starts, succeeds or fails, the running, success or error handler, respectively, will pull a list of relevant notifications using the procedure defined above. It then creates a Notification Object for each one containing relevant details about the job and then **sends** it to the destination (email addresses, Slack channel(s), SMS numbers, etc.). These Notification objects are available as related resources on job types (Jobs, Inventory Updates, Project Updates), and also at `/api/v2/notifications`. You may also see what notifications have been sent by examining its related resources.

Notifications can succeed or fail but that will _not_ cause its associated job to succeed or fail. The status of the notification can be viewed at its detail endpoint: `/api/v2/notifications/<n>`

## Testing Notifications Before Using Them

Once a Notification Template is created, its configuration can be tested by utilizing the endpoint at `/api/v2/notification_templates/<n>/test`   This will emit a test notification given the configuration defined by the notification.  These test notifications will also appear in the notifications list at `/api/v2/notifications`

# Notification Types

The currently defined Notification Types are:

* Email
* Slack
* HipChat
* Mattermost
* Rocket.Chat
* Pagerduty
* Twilio
* IRC
* Webhook
* Grafana

Each of these have their own configuration and behavioral semantics and testing them may need to be approached in different ways. The following sections will give as much detail as possible.


## Email

The email notification type supports a wide variety of SMTP servers and has support for SSL/TLS connections and timeouts.

### Testing Considerations

The following should be performed for good acceptance:

* Test plain authentication.
* Test SSL and TLS authentication.
* Verify single and multiple recipients.
* Verify message subject and contents are formatted sanely.  They should be plaintext but readable.

### Test Service

Set up a local SMTP mail service.  Some options are listed below:

* Postfix service on galaxy: https://galaxy.ansible.com/debops/postfix/
* Mailtrap has a good free plan that should provide all of the features necessary: https://mailtrap.io/
* Another option is to use a Docker container: `docker run --network="tools_default" -p 25:25 -e maildomain=mail.example.com -e smtp_user=user:pwd --name postfix -d catatnight/postfix`


## Slack

Slack is simple to configure; it requires a token, which you can get from creating a bot in the integrations settings for the Slack team.

### Testing Considerations

The following should be performed for good acceptance:

* Test single and multiple channels and good formatting of the message. Note that slack notifications only contain the minimal information.


## Mattermost

The Mattermost notification integration uses Incoming Webhooks. A password is not required because the webhook URL itself is the secret. Webhooks must be enabled in the System Console of Mattermost. If the user wishes to allow Ansible Tower notifications to modify the Icon URL and username of the notification, then they must enabled these options as well.

In order to enable these settings in Mattermost:
1. Go to System Console > Integrations > Custom Integrations. Check "Enable Incoming Webhooks".
2. Optionally, go to System Console > Integrations > Custom Integrations. Check "Enable integrations to override usernames" and Check "Enable integrations to override profile picture icons".
3. Go to Main Menu > Integrations > Incoming Webhook. Click "Add Incoming Webhook".
4. Choose a "Display Name", "Description", and Channel. This channel will be overridden if the notification uses the `channel` option.

* `url`: This is the incoming webhook URL that was configured in Mattermost. Notifications will use this URL to `POST`.
* `username`: Optional. The username to display for the notification.
* `channel`: Optional. Override the channel in which to display the notification. Mattermost incoming webhooks are tied to a channel by default, so if left blank then this will use the incoming webhook channel. Note, if the channel does not exist, then the notification will error out.
* `icon_url`: Optional. A URL pointing to an icon to use for the notification.

### Testing Considerations

* Make sure all options behave as expected.
* Test that all notification options are obeyed.
* Test formatting and appearance. Mattermost will use the minimal version of the notification.

### Test Service

* Utilize an existing Mattermost installation or use their Docker container here: `docker run --name mattermost-preview -d --publish 8065:8065 mattermost/mattermost-preview`
* Turn on Incoming Webhooks and optionally allow Integrations to override usernames and icons in the System Console.


## Rocket.Chat

The Rocket.Chat notification integration uses Incoming Webhooks. A password is not required because the webhook URL itself is the secret. An integration must be created in the Administration section of the Rocket.Chat settings.

The following fields are available for the Rocket.Chat notification type:
* `url`: The incoming webhook URL that was configured in Rocket.Chat. Notifications will use this URL to `POST`.
* `username`: Optional. Change the displayed username from Rocket.Chat to specified username.
* `icon_url`: Optional. A URL pointing to an icon to use for the notification.

### Testing Considerations

* Make sure that all options behave as expected.
* Test that all notification options are obeyed.

### Test Service

* Utilize an existing Rocket.Chat installation or use their Docker containers from https://rocket.chat/docs/installation/docker-containers/
* Create an Incoming Webhook in the Integrations section of the Administration settings


## Pagerduty

Pagerduty is a fairly straightforward integration. The user will create an API Key in the Pagerduty system (this will be the token that is given to Tower) and then create a "Service" which will provide an "Integration Key" that will also be given to Tower. The other options of note are:

* `subdomain`: When you sign up for the Pagerduty account, you will get a unique subdomain to communicate with. For instance, if you signed up as "towertest", the web dashboard will be at *towertest.pagerduty.com* and you will give the Tower API "towertest" as the subdomain (not the full domain).
* `client_name`: This will be sent along with the alert content to the Pagerduty service to help identify the service that is using the API key/service.  This is helpful if multiple integrations are using the same API key and service.

### Testing considerations

* Make sure the alert lands on the Pagerduty service.
* Verify that the minimal information is displayed for the notification but also that the detail of the notification contains all fields. Pagerduty itself should understand the format in which we send the detail information.

### Test Service

Pagerduty allows you to sign up for a free trial with the service.


## Twilio

Twilio is a Voice and SMS automation service.  Once you are signed in, you'll need to create a phone number from which the message will be sent.  You'll then define a "Messaging Service" under Programmable SMS and associate the number (the one you created for this purpose) with it.  Note that you may need to verify this number or some other information before you are allowed to use it to send to any numbers. The Messaging Service does not need a status callback URL nor does it need the ability to process inbound messages.

Under your individual (or sub) account settings, you will have API credentials. The Account SID and AuthToken are what will be given to Tower. There are a couple of other important fields:

* `from_number`:  This is the number associated with the messaging service above and must be given in the form of "+15556667777".
* `to_numbers`: This will be the list of numbers to receive the SMS and should be the 10-digit phone number.

### Testing Considerations

* Test notifications with single and multiple recipients.
* Verify that the minimal information is displayed for the notification.  Note that this notification type does not display the full detailed notification.

### Test Service

Twilio is fairly straightforward to sign up for but there may not be a free trial offered; a credit card will be needed to sign up for it though the charges are fairly minimal per message.


## IRC

The Tower IRC notification takes the form of an IRC bot that will connect, deliver its messages to channel(s) or individual user(s), and then disconnect.  The Tower notification bot also supports SSL authentication.  The Tower bot does not currently support Nickserv identification.  If a channel or user does not exist or is not online, then the Notification will not fail; the failure scenario is reserved specifically for connectivity.

Connectivity information is straightforward:

* `server`: The host name or address of the IRC server.
* `port`: The IRC server port.
* `nickname`: The bot's nickname once it connects to the server.
* `password`: IRC servers can require a password to connect.  If the server doesn't require one, then this should be an empty string.
* `use_ssl`: If you want the bot to use SSL when connecting.
* `targets`: A list of users and/or channels to send the notification to.

### Test Considerations

* Test both plain and SSL connectivity.
* Test single and multiples of both users and channels.

### Test Service

There are a few modern IRC servers to choose from.  [InspIRCd](http://www.inspircd.org/) is recommended because it is actively maintained and pretty straightforward to configure.


## Webhook

The webhook notification type in Ansible Tower provides a simple interface to sending `POST`s to a predefined web service.  Tower will `POST` to this address using `application/json` content type with the data payload containing all relevant details in json format.

The parameters are fairly straightforward:

* `url`: The full URL that will be `POST`ed to
* `headers`: Headers in json form where the keys and values are strings. For example: `{"Authentication": "988881adc9fc3655077dc2d4d757d480b5ea0e11", "MessageType": "Test"}`

### Test Considerations

* Test HTTP service and HTTPS, also specifically test HTTPS with a self signed cert.
* Verify that the headers and payload are present, that the payload is json, and the content type is specifically `application/json`

### Test Service

A very basic test can be performed by using `netcat`:

```
netcat -l 8099
```

...and then sending the request to: *http://\<host\>:8099*

Note that this won't respond correctly to the notification, so it will yield an error. Using a very basic Flask application for verifying the `POST` request is recommended; you can see an example here:

https://gist.github.com/matburt/73bfbf85c2443f39d272

The link below shows how to define an endpoint and parse headers and json content. It doesn't show how to configure Flask for HTTPS, but is fairly straightforward:
http://flask.pocoo.org/snippets/111/

You can also link an `httpbin` service to the development environment for testing webhooks using:

```
docker run --network="tools_default" --name httpbin -p 8204:80 kennethreitz/httpbin
```

This will create an `httpbin` service reachable from the AWX container at `http://httpbin/post`, `http://httpbin/put`, etc. Outside of the container, you can reach the service at `http://localhost:8204`.


## Grafana

The Grafana notification type allows you to create Grafana annotations. Details about this feature of Grafana are available at http://docs.grafana.org/reference/annotations/. In order to allow Tower to add annotations, an API Key needs to be created in Grafana. Note that the created annotations are region events with start and endtime of the associated Tower Job. The annotation description is also provided by the subject of the associated Tower Job, for example:

```
Job #1 'Ping Macbook' succeeded: https://towerhost/#/jobs/playbook/1
```

The configurable options of the Grafana notification type are:
* `Grafana URL`: Required. The base URL of the Grafana server. **Note**: the `/api/annotations` endpoint will be added automatically to the base Grafana URL.
* `API Key`: Required. The Grafana API Key to authenticate.
* `ID of the Dashboard`: Optional. To create annotations in a specific Grafana dashboard, enter the ID of the dashboard.
* `ID of the Panel`: Optional. To create annotations in a specific Panel, enter the ID of the panel.
**Note**: If neither `dashboardId` nor `panelId` are provided, then a global annotation is created and can be queried in any dashboard that adds the Grafana annotations data source.
* `Annotations tags`: The list of tags to add to the annotation. One tag per line.
* `Disable SSL Verification`: Disable the verification of the SSL certificate, _e.g._, when using a self-signed SSL certificate for Grafana.

### Test Considerations

* Make sure that all options behave as expected.
* Test that all notification options are obeyed.
* Make sure the annotation gets created on the desired dashboard and/or panel and with the configured tags.

### Test Service
* Utilize an existing Grafana installation or use their Docker containers from http://docs.grafana.org/installation/docker/.
* Create an API Key in the Grafana configuration settings.
* (Optional) Lookup `dashboardId` and/or `panelId` if needed.
* (Optional) Define tags for the annotation.