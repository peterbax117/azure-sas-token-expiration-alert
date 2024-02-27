# Azure Storage Account Shared Access Signature (SAS) Token Alert Solution

Create a simple alert solution for a shared access signature (SAS) token in an Azure storage account.  This solution uses a Key Vault to manage the Alert as the Storage Account does not offer an option to alert for SAS expiration due to the design of the product.  We will then use several Events that are avaialble in Key Vault to send an email alert to a user or distribution group.

We will assume you already have the SAS token and know the expiration date of the token.  We will show you how to create the Key Vault and then setup the Action Group for alerting.  We will also add the SAS token as a Secret within the vault.  Then we will setup monitoring of certain Events related to expiration date for the Secret you created.

This involves using Secrets store and the following Events that are avaible in the Key Vault:

* _Microsoft.KeyVault.SecretNearExpiry_
* _Microsoft.KeyVault.SecretExpired_

> [!NOTE]
> _SecretNearExpiry_ is triggered when the current version of a secret is about to expire. (The event is ***triggered 30 days before*** the expiration date.)  _SecretExpired_ is triggered when the current version of a secret is expired.

Reference: [Azure Key Vault as Event Grid source](https://learn.microsoft.com/en-us/azure/event-grid/event-schema-key-vault?tabs=event-grid-event-schema#available-event-types)

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

## Create Event Subscriptions for Secret Expiring and Secret Expired

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

