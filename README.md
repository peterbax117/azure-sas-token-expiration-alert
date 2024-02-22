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

![visual](/images/01-KV-Create.png)

Fill in the Basic information about the vault. I am keeping the default settings for _Days to retain deleted vaults_ and _Purge protection_ as well as the Standard _Pricing tier_.  Choose _Next_

![visual](/images/02-KV-Basic-Info.png)

For Access Configuration, use the default settings.  You do not need to check any of the boxes for _Resource access_. 
Choose _Next_

![visual](/images/03-KV-Access.png)

For Networking, adjust settings based on the security requirements of your organization.  As an example, most of the customers I work with require private endpoints for any Azure resource.  Becasue we are using the Key Vault as a self contained repository, access to it other than from the Azure portal is not required.  If you wanted to scale this solution or automate the entry of the Secret into the Key Vault then 
