# Azure Storage Account Shared Access Signature (SAS) Token Expiring Alert Solution

Create a simple alert solution for a shared access signature (SAS) token in an Azure storage account.  This solution uses a Key Vault to manage the Alert as the Storage Account does not offer an option to alert for SAS expiration due to the design of the product.  We will then use several Events that are available in Key Vault to send an email alert to a user or distribution group.

* An example of a SAS Token for accessing a Storage Account:
* sv=2022-11-02&ss=bf&srt=sco&sp=rwdlaciyx&se=2024-03-01T02:00:00Z&st=2024-02-09T18:44:32Z&spr=https&sig=3bRdnoOoCcZVXZ6X2a8IUaKrDFJnSDrkJQcRZuvbFto%3D

We will assume you already have the SAS token and know the expiration date of the token.  We will show you how to create the Key Vault and then setup the Action Group for alerting.  We will also add the SAS token as a Secret within the vault.  Then we will setup monitoring of certain Events related to expiration date for the Secret you created.

This involves using Secrets store and the following Events that are avaible in the Key Vault:

* _Microsoft.KeyVault.SecretNearExpiry_
* _Microsoft.KeyVault.SecretExpired_
     
> [!NOTE]
> _SecretNearExpiry_ is triggered when the current version of a secret is about to expire. (The event is ***triggered 30 days before*** the expiration date.)  _SecretExpired_ is triggered when the current version of a secret is expired.
   
Reference: [Azure Key Vault as Event Grid source](https://learn.microsoft.com/en-us/azure/event-grid/event-schema-key-vault?tabs=event-grid-event-schema#available-event-types)

Table of Contents

[Create the Azure Key Vault](../main/README.md#create-the-azure-key-vault)  
[Create an Action Group in Azure Monitor](../main/README.md#create-an-action-group-in-azure-monitor)  
[Create Event Subscription for Secret Near Expiry Event](../main/README.md#create-event-subscription-for-secret-near-expiry-event)  
[Create Event Subscription for Secret Expired](../main/README.md#create-event-subscription-for-secret-expired)  
[Create the Secret in the Key Vault](../main/README.md#create-the-secret-in-the-key-vault)  
[Email Examples for Secret Expiry and Secret Expired](../main/README.md#email-examples-for-secret-expiry-and-secret-expired)  

## Create the Azure Key Vault

I would recommend creating a new Key Vault for managing these Alerts.  The cost is minimal and can be kept in a central management subscription that other applications or workloads could use.

First, search for "_Key Vault_" in the Azure Portal and then open the Key Vault interface.  

Choose _+ Create_

![visual](/images/2024-02-KV-01-Create.png)

Fill in the Basic information about the vault. I am keeping the default settings for _Days to retain deleted vaults_ and _Purge protection_ as well as the Standard _Pricing tier_.  Choose _Next_

![visual](/images/2024-02-KV-02-Basic-Info.png)

For Access Configuration, use the default settings.  You do not need to check any of the boxes for _Resource access_. 
Choose _Next_

![visual](/images/2024-02-KV-03-Access.png)

For Networking, adjust settings based on the security requirements of your organization.  As an example, most of the customers I work with require private endpoints for any Azure resource.  Becasue we are using the Key Vault as a self contained repository, access to it other than from the Azure portal is not required.  If you wanted to scale this solution or automate the entry of the Secret into the Key Vault then you would need to consider how the Key Vault is accessed.

![visual](/images/2024-02-KV-04-Network.png)

Review all of the configuration settings and then click _Create_

![visual](/images/2024-02-KV-05-Review.png)

The Key Vault should be created and deployed and you should be able to access it.  Keep in mind that in order to enter a Secret into the vault you will need to have the proper permissions.  Owner of the Key Vault is not sufficient to create or manage secrets.  One of the following roles will be needed to create or manage Secrets directly:

* Key Vault Administrator
* Key Vault Secrets Officer


## Create an Action Group in Azure Monitor

You or your organization may already have Action Groups setup in Azure Monitor.  If not, we will create one now to send the email alert to when the Events fire.

First, search for "_Azure Monitor_" in the Azure Portal and then open the Azure Monitor interface. In the Azure Monitor interface choose _Alerts_ in the left navigation menu.  Then choose _Action Groups_ in the upper menu.

![visual](/images/2024-02-AG-01-Monitor.png)

In the Action Group menu, choose _+ Create_

![visual](/images/2024-02-AG-02-Create.png)

In the Create action group form, fill in the basic information.  Region should be kept as Global.  Keep in mind that the Display Name has a 12 character limit.

Choose _Next_

![visual](/images/2024-02-AG-03-Basics.png)

For Notifications, choose _Email/SMS message/Push/Voice_.  Give it a name like "Notify me by email." 

Click the _Edit_ button

![visual](/images/2024-02-AG-04-Notifications.png)

For this example I am going to send an email to myself.  You can choose any type of notification that makes sense for you and your organization.  Once you have chosen the notification type and settings, choose _"Yes"_ for _Enable the common alert schema_.

Click _OK_

![visual](/images/2024-02-AG-05-Email.png)

Review all of the configuration settings and then click _Create_

![visual](/images/2024-02-AG-06-Review.png)

## Create Event Subscription for Secret Near Expiry Event

We are going to create 2 different event subscriptions.  The first will be for when the event SecretExpiryNear is fired and the second for when the event SecretExpired is fired.  When these events fire an email will be sent to a user or distribution group.  In the case of this example I will be sending an email to myself.

Go to the Key Vault created earlier and click on the Events in the left navigation menu.

Choose _+ Event Subscription_

![visual](/images/2024-02-EV-01-Event-Start.png)

Enter a name for the Event Subscription.  Choose the Event Schema.  Create or choose a System Topic Name.

* Event Subscription: "SAS-Token-Expiring" (You can choose whatever name you like here.  Be aware this will be part of the email sent for the alert so consider that when naming)
* Event Schema: Cloud Event Schema v1.0
* System Topic: "sas-topic-kv" (This can be anything you want.  Just create a topic that makes sense here or follows your organization guidelines.)

![visual](/images/2024-02-EV-02-Basics.png)

By default all 9 Event Types will be selected

![visual](/images/2024-02-EV-03-Type-Default.png)

For this Event we will choose _Secret Near Expiry_ only.

![visual](/images/2024-02-EV-04-Type-Near.png)

![visual](/images/2024-02-EV-05-Type-Expiry.png)

For the Enpoint Type we will choose _Azure Monitor Alert_

![visual](/images/2024-02-EV-06-Endpoint.png)

We then need to configure the endpoint.  Click on the _Configure an enpoint_ link.

![visual](/images/2024-02-EV-07-Endpoint-Configure.png)

In the ***Select Monitor Alert Configuration*** form that opens, for Alert Severity, choose _Sev 1 (Error)_(You can choose what you want here, but this made the most sense for me when creating this alert).

Check the box for _Select action groups_.  Then choose the Action Group you created earlier or choose one that already exsists that you want to send the alert to.

![visual](/images/2024-02-EV-08-Endpoint-AG.png)

Create an _Alert description_.  Consider that this description will be in the email sent when the alert fires.  If you are using email rules or some other type of automation you may want to key off of this description.

Here is what I used:

* A SAS Token will expire in the next 30 days.  Please review.

![visual](/images/2024-02-EV-09-Endpoint-Description.png)

Click _Confirm Selection_ button to finish configuring the endpoint.

![visual](/images/2024-02-EV-10-Endpoint-Confirm.png)

We will now come back to the Basics section for the Event and can click the _Create_ button as all the sections should be shown as filled out.  We will not use any of the other sections such as Filters, Additional Features, Delivery Properties, or Advanced Editor.

![visual](/images/2024-02-EV-11-Event-Create.png)

We now have a working Event Subscription

## Create Event Subscription for Secret Expired

We will now create a second event for when the Secret is Expired.  In this case, I am going to only call out the differences needed for this Event Subscription.

Give the Event a name and choose the Schema, and then reuse the System Topic Name from above

* Event Name: SAS-Token-Expired
* Event Schema: Cloud Event Schema v1.0

![visual](/images/2024-02-EV-12-Event-Expired.png)

For the _Event Type_ we will choose ***Secret Expired***

![visual](/images/2024-02-EV-13-Type-Expired.png)

When setting up the Azure Monitor endpoint configuration make the following changes:

* Alert severity: Sev 0 (Critical)
* Check _Select action groups_: Choose the action group from above or relelvant group for your organization
* Alert description: A SAS Token has EXPIRED! Please ACT NOW!

![visual](/images/2024-02-EV-14-Alert-Expired.png)

## Create the Secret in the Key Vault

In the Key Vault, choose Secrets in the left navigation

Click _+ Generate/Import_

![visual](/images/2024-02-S-01-Generate.png)

In the _Create a secret_ form we will fill out all the basic information related to the SAS Token.  What I used is listed below.  2 Items of note, first, you can store the SAS Token in the Secret value field, but this is not required.  You can technically put any value you want in this field.  Second, when you set the expiration date pay close attendtion to the year.  The date box will choose a date 2 years into the future by default.  Most SAS Token scenarios will be for 2 years to be sure to choose the correct year for the expiration.

* Upload options: Manual
* Name: SAS-Token-Petetoso-Corp
* Secret value: sv=2022-11-02&ss=bf&srt=sco&sp=rwdlaciyx&se=2024-03-01T02:00:00Z&st=2024-02-09T18:44:32Z&spr=https&sig=3bRdnoOoCcZVXZ6X2a8IUaKrDFJnSDrkJQcRZuvbFto%3D
* Set expiration date: Checked
* Expiration date: 02/28/2024 06:00:00 PM
* Enabled: Yes

![visual](/images/2024-02-S-02-Basic.png)

Then click _Create_ to save the form

![visual](/images/2024-02-S-03-Save.png)

You will then be returned to the main Secret page.  Verify that you have inputed the correct expiration date.

![visual](/images/2024-02-S-04-Verify.png)

## Email Examples for Secret Expiry and Secret Expired

These are examples of the emails you will receive when a Secret is near expiry or has expired.  Remember that near expiry is within the next 30 days.  For testing purposes the emails will not fire off instantly.  In my testing it can take as long as 15 minutes to receive an email.  You can use certain areas to filter the emails such as wording in the Subject or Body of the message or feed them email into another automation system to then be further utilized.

### Near Expiry

The Subject contains the term ***Fired***, then ***Severity***, then ***Azure Monitor Alert***, then the ***Name*** of the alert (SAS-TOKEN-EXPIRING), then in which ***Key Vault***.

In the upper section of the body of the email is the Alert Name, Severity, name of the Key Vault, Description.

![visual](/images/2024-02-EM-01-ExpiryPart1.png)

In the lower section of the body of the email, the Event Subject and Event Type are the important components.  Event Subject listing the name of the customer this would affect as the customer name was used in the name of the Secret.

![visual](/images/2024-02-EM-02-ExpiryPart2.png)

### Expired

In the upper section of the body of the email is the Alert Name, Severity, name of the Key Vault, Description.

![visual](/images/2024-02-EM-03-ExpiredPart1.png)

In the lower section of the body of the email, the Event Subject and Event Type are the important components.  Event Subject listing the name of the customer this would affect as the customer name was used in the name of the Secret.

![visual](/images/2024-02-EM-04-ExpiredPart2.png)
