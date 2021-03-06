AzineMailgunWebhooksBundle
==========================

Symfony2 Bundle to capture event data from the Mailgun.com transactional mail service.

If you are using a free mailgun.com account (less than 10'000 email per month), then
mailgun.com deletes log-entries about events after about 48 hours. 

So if you want to check who recieved the newsletter you sent last week, your busted. :-(

Mailgun.com offeres the cool feature to post the event data to an URL of your choice.
=> see http://documentation.mailgun.com/user_manual.html#webhooks for more details.

This bundle captures this data. You can search for, filter and display log-entries and
delete them when you don't need them anymore (or when you need to save some disk-space).  


## Features
- capture all data that mailgun.com can post via the "webhooks" provided by mailgun.com => http://documentation.mailgun.com/user_manual.html#webhooks
- display lists of event entries with search and filter functionality
- show all details of a singel event
- cli command to delete events 'old' events
- cli command to check if mailguns sending ip address is on any block-list, including email notification to administrator if the IP is on a black-list.
- email notification to administrator when mails bounce because of a SPAM rating
 

## Installation
To install AzineMailgunWebhooksBundle with Composer just add the following to your `composer.json` file:

```
// composer.json
{
    // ...
    require: {
        // ...
        "azine/mailgunwebhooks-bundle": "dev-master",
    }
}
```
Then, you can install the new dependencies by running Composer’s update command from 
the directory where your `composer.json` file is located:

```
php composer.phar update
```
Now, Composer will automatically download all required files, and install them for you. 
All that is left to do is to update your AppKernel.php file, and register the new bundle:

```
<?php

// in AppKernel::registerBundles()
$bundles = array(
    // ...
    new Azine\MailgunWebhooksBundle\AzineMailgunWebhooksBundle(),
    // ...
);
```

Register the routes of the AzineMailgunWebhooksBundle:

```
// in app/config/routing.yml

# Route for mailgun.com to post the information that we want to store in the database
azine_mailgun_webhooks_bundle_webhook:
    resource: "@AzineMailgunWebhooksBundle/Resources/config/routing/mailgunevent_webhook.yml"
    prefix:   /

# Routes for administration of the posted data
azine_mailgun_webhooks_bundle_admin:
    resource: "@AzineMailgunWebhooksBundle/Resources/config/routing/mailgunevent_admin.yml"
    prefix:   /admin/
    
```

## Configuration options
This is the complete list of configuration options with their defaults.
```
// app/config/config.yml
# Default configuration for "AzineMailgunWebhooksBundle"
azine_mailgun_webhooks:

    # Your api-key for mailgun => see https://mailgun.com/cp
    api_key:              ~ # Required

    # Your public-api-key for mailgun => see https://mailgun.com/cp
    public_api_key:       ''
    
    spam_alerts:
    
        # Whether to send notifications about spam complaints
        enabled:   false
        
        #Interval in minutes between sending of email notifications after receiving spam complaints
        interval:  '60'
        
        # Mailgun helpdesk ticket ID to request new IP address in case of spam complains
        ticket_id:      ''
        
        #Mailgun HelpDesk ticket subject
        ticket_subject: 'IP on spam-list, please fix.'
        
        #Mailgun HelpDesk ticket subject
        ticket_message: 'It looks like my ip is on a spam-list. Please, assign a clean IP to my domain."
        
        # Admin E-Mail to send notification about spam complaints
        alerts_recipient_email:     ''
        
    hetrixtools_service:
    
        #Your public-api-key for hetrixtools => see https://hetrixtools.com/
        api_key:        ''
        
        #Url for checking if ip is in blacklist => see https://docs.hetrixtools.com/blacklist-check-api/
        blacklist_check_ip_url:    'https://api.hetrixtools.com/v2/<API_TOKEN>/blacklist-check/ipv4/<IP_ADDRESS>/'
            
```

# Sidenote on "monolog" emails and web scanners
You can configure monolog to send emails whenever an error occurs.
=> http://symfony.com/doc/current/cookbook/logging/monolog_email.html 

It is likely that many 404-errors occur on you site because web-scanners 
try to see if you are hosting vulnerable scripts on your server. If 
these errors are mailed via mailgun.com as well, you might send a lot more 
mails than you want to (and exceed the limit of 10k free emails) and it  
will clutter you database with more or less useless information.

Since Symfony 2.4, to avoid these emails being sent, you can configure 
monolog to ignore certain 404 errors.
=> http://symfony.com/doc/current/cookbook/logging/monolog_regex_based_excludes.html  

```
// app/config/config.yml
monolog:
    handlers:
        main:
            type:         fingers_crossed
            action_level: warning
            handler:      yourNextHandler
            excluded_404s:
                - ".*/cgi-bin/php.*"
                - ".*MyAdmin/scripts/setup.php.*"
                - ".*autoconfig/mail/config-v1.1.xml.*"
                - ".*vtigercrm/graph.php.*"
                - ".*/HNAP1/.*"
                - ".*calendar/install/index.php.*"
                - ".*admin/config.php.*"
```

# Webhooks configuration of mailgun.com
To tell mailgun.com to post the data to your database via the webhooks, just
get the full url of the "mailgunevent_webhook"-route

```
# on a bash console execute this to get the absolute webhook path
php bin/console debug:router -e prod | grep mailgunevent_webhook 
// note for Symfony 2.x it is 'php app/console debug:router -e prod | grep mailgunevent_webhook'

```

and copy it to all the input fields for the webhooks on https://mailgun.com/app/webhooks

Then test if everything is setup ok by clicking the "Test" or "Send" button and check
you database or the event-list.

```
# on a bash console execute this to get the absolute overview-page path
php bin/console router:debug -e prod | grep mailgun_overview
// note for Symfony 2.x it is 'php app/console debug:router -e prod | grep mailgun_overview'
```

## Events
Whenever mailgun posts an event via the webhook, an MailgunWebhookEvent containing the 
new MailgunEvent is dispatched.

You can implement your own means of notification for failures or if you configured your
application to use the swiftmailer, you can use the SwiftMailerMailgunWebhookEventListener,
to send emails to an address you specified.

# Cli Commands
This bundle offers two commands for you to automate things via a scheduler (cronjob).

## Delete old MailgunEvents
If you only want to keep mailgun events younger than `date string`, to prevent your database of running full, you can run the command via scheduler/cronjob. 

```
# e.g. delete all events older than 60 days
php app/console mailgun:delete-events -date '60 days ago'
```

## Check if Sending IP is on Black-Lists
If the IP address, your emails are sent from, gets listed on a SPAM block-list, your email deliverability drops drastically. 
To be able to be notified a.s.a.p. you can run this command to check the last IP, that was used for sending your emails, against
hetrixtools.com. If the IP is listed, the administrator will receive an email.

```
# e.g. delete all events older than 60 days
php app/console mailgun:check-ip-in-blacklist -numberOfAttempts 5
```
With the parameter `numberOfAttempts` you can specify how many times the command should try to get the hetrixtools report, if an attempt fails for any reason.

HetrixTools.com has a free, limited plan that will allow ~3 checks per day. See https://hetrixtools.com/pricing/blacklist-monitor/ for details.
  

# ToDos / Contribute
Anyone is welcome to contribute.

Here's a list of open TODOs
- write more unit-tests
- add commands to "cleanup" the database periodically
- add SwiftMailerMailgunWebhookEventListener to notify admins when certain events occur => email upon "dropped" event
- write some CSS, style pages 

## Build-Status ec.

[![Build Status](https://travis-ci.org/azine/AzineMailgunWebhooksBundle.png)](https://travis-ci.org/azine/AzineMailgunWebhooksBundle)
[![Total Downloads](https://poser.pugx.org/azine/mailgunwebhooks-bundle/downloads.png)](https://packagist.org/packages/azine/mailgunwebhooks-bundle)
[![Latest Stable Version](https://poser.pugx.org/azine/mailgunwebhooks-bundle/v/stable.png)](https://packagist.org/packages/azine/mailgunwebhooks-bundle)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/azine/AzineMailgunWebhooksBundle/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/azine/AzineMailgunWebhooksBundle/?branch=master)
[![Code Coverage](https://scrutinizer-ci.com/g/azine/AzineMailgunWebhooksBundle/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/azine/AzineMailgunWebhooksBundle/?branch=master)
