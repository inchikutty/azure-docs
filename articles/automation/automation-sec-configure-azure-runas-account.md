---
title: Configure Azure Run As Account | Microsoft Docs
description: Tutorial that walks you through the creation, testing, and example use of security principal authentication in Azure Automation.
services: automation
documentationcenter: ''
author: mgoedtel
manager: carmonm
editor: ''
keywords: service principal name, setspn, azure authentication

ms.assetid: 2f783441-15c7-4ea0-ba27-d7daa39b1dd3
ms.service: automation
ms.workload: tbd
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: get-started-article
ms.date: 03/15/2017
ms.author: magoedte

---
# Authenticate Runbooks with Azure Run As account
This topic will show you how to configure an Automation account from the Azure portal using the Run As account feature to authenticate runbooks managing resources in either Azure Resource Manager or Azure Service Management.

When you create a new Automation account in the Azure portal, it automatically creates:

* Run As account which creates a new service principal in Azure Active Directory, a certificate, and assigns the Contributor role-based access control (RBAC), which will be used to manage Resource Manager resources using runbooks.   
* Classic Run As account by uploading a management certificate, which will be used to manage Azure Service Management or classic resources using runbooks.  

This simplifies the process for you and helps you quickly start building and deploying runbooks to support your automation needs.      

Using a Run As and Classic Run As account, you can:

* Provide a standardized way to authenticate with Azure when managing Azure Resource Manager or Azure Service Management resources from runbooks in the Azure portal.  
* Automate the use of global runbooks configured in Azure Alerts.

> [!NOTE]
> The Azure [Alert integration feature](../monitoring-and-diagnostics/insights-receive-alert-notifications.md) with Automation Global Runbooks requires an Automation account that is configured with a Run As and Classic Run As account. You can either select an Automation account that already has a Run As and Classic Run As account defined or choose to create a new one.
>  

We will show you how to create the Automation account from the Azure portal, update an Automation account using PowerShell, manage the account configuration, and demonstrate how to authenticate in your runbooks.

Before we do that, there are a few things that you should understand and consider before proceeding.

1. This does not impact existing Automation accounts already created in either the classic or Resource Manager deployment model.  
2. This will only work for Automation accounts created through the Azure portal.  Attempting to create an account from the classic portal will not replicate the Run As account configuration.
3. If you currently have runbooks and assets (i.e. schedules, variables, etc.) previously created to manage classic resources, and you want those runbooks to authenticate with the new Classic Run As account, you will need to create a Classic Run As Account using Managing an Run As Account or update your existing account using the PowerShell script below.  
4. To authenticate using the new Run As account and Classic Run As Automation account, you will need to modify your existing runbooks with the example code below.  **Please note** that the Run As account is for authentication against Resource Manager resources using the certificate-based service principal, and the Classic Run As account is for authenticating against Service Management resources with the management certificate.     

## Create a new Automation Account from the Azure portal
In this section, you will perform the following steps to create a new Azure Automation account  from the Azure portal.  This creates both the Run As and classic Run As account.  

> [!NOTE]
> The user performing these steps must be a member of the Service Admins role or co-administrator of the subscription which is granting access to the subscription for the user. The user must also be added as a User to that subscriptions default Active Directory; the account does not need to be assigned to a privileged role. Users who are not a member of the Subscription’s Active Directory prior to being added to the Co-Admin role of the subscription will be added to Active Directory as a Guest and will see the “You do not have permissions to create…” warning in the **Add Automation Account** blade. Users who were added to the co-admin role first can be removed from the subscriptions Active Directory and re-added to make them a full User in Active Directory. This situation can be verified from the **Azure Active Directory** pane in the Azure portal by selecting **Users and groups**, select **All users** and after selecting the specific user select **Profile**.  The value of the **User type** attribute under the users profile should not equal **Guest**.  
> 

1. Sign in to the Azure portal with an account that is a member of the Subscription Admins role and co-administrator of the subscription.
2. Select **Automation Accounts**.
3. In the Automation Accounts blade, click **Add**.<br>![Add Automation Account](media/automation-sec-configure-azure-runas-account/create-automation-account-properties-b.png)
   
   > [!NOTE]
   > If you see the following warning in the **Add Automation Account** blade, this is because your account is not a member of the Subscription Admins role and co-admin of the subscription.<br>![Add Automation Account Warning](media/automation-sec-configure-azure-runas-account/create-account-without-perms.png)
   > 
   > 
4. In the **Add Automation Account** blade, in the **Name** box type in a name for your new Automation account.
5. If you have more than one subscription, specify one for the new account, as well as a new or existing **Resource group** and an Azure datacenter **Location**.
6. Verify the value **Yes** is selected for the **Create Azure Run As account** option, and click the **Create** button.  
   
   > [!NOTE]
   > If you choose to not create the Run As account by selecting the option **No**, you will be presented with a warning message in the **Add Automation Account** blade.  While the account is created in the Azure portal, it will not have a corresponding authentication identity within your classic or Resource Manager subscription directory service and therefore, no access to resources in your subscription.  This will prevent any runbooks referencing this account from being able to authenticate and perform tasks against resources in those deployment models.
   > 
   > ![Add Automation Account Warning](media/automation-sec-configure-azure-runas-account/create-account-decline-create-runas-msg.png)<br>
   > When the service principal is not created the Contributor role will not be assigned.
   > 

7. While Azure creates the Automation account, you can track the progress under **Notifications** from the menu.

### Resources included
When the Automation account is successfully created, several resources are automatically created for you.  The following table summarizes resources for the Run As account.<br>

| Resource | Description |
| --- | --- |
| AzureAutomationTutorial Runbook |An example Graphical runbook that demonstrates how to authenticate using the Run As account and gets all the Resource Manager resources. |
| AzureAutomationTutorialScript Runbook |An example PowerShell runbook that demonstrates how to authenticate using the Run As account and gets all the Resource Manager resources. |
| AzureRunAsCertificate |Certificate asset automatically created during Automation account creation or using the PowerShell script below for an existing account.  It allows you to authenticate with Azure so that you can manage Azure Resource Manager resources from runbooks.  This certificate has a one-year lifespan. |
| AzureRunAsConnection |Connection asset automatically created during Automation account creation or using the PowerShell script below for an existing account. |

The following table summarizes resources for the Classic Run As account.<br>

| Resource | Description |
| --- | --- |
| AzureClassicAutomationTutorial Runbook |An example Graphical runbook which gets all the Classic VMs in a subscription using the Classic Run As Account (certificate) and then outputs the VM name and status. |
| AzureClassicAutomationTutorial Script Runbook |An example PowerShell runbook which gets all the Classic VMs in a subscription using the Classic Run As Account (certificate) and then outputs the VM name and status. |
| AzureClassicRunAsCertificate |Certificate asset automatically created that is used to authenticate with Azure so that you can manage Azure classic resources from runbooks.  This certificate has a one-year lifespan. |
| AzureClassicRunAsConnection |Connection asset automatically created that is used to authenticate with Azure so that you can manage Azure classic resources from runbooks. |

## Verify Run As authentication
Next we will perform a small test to confirm you are able to successfully authenticate using the new Run As account.     

1. In the Azure portal, open the Automation account created earlier.  
2. Click on the **Runbooks** tile to open the list of runbooks.
3. Select the **AzureAutomationTutorialScript** runbook and then click **Start** to start the runbook.  You will receive a prompt verifying you wish to start the runbook.
4. A [runbook job](automation-runbook-execution.md) is created, the Job blade is displayed, and the job status displayed in the **Job Summary** tile.  
5. The job status will start as *Queued* indicating that it is waiting for a runbook worker in the cloud to become available. It will then move to *Starting* when a worker claims the job, and then *Running* when the runbook actually starts running.  
6. When the runbook job completes, we should see a status of **Completed**.<br> ![Security Principal Runbook Test](media/automation-sec-configure-azure-runas-account/job-summary-automationtutorialscript.png)<br>
7. To see the detailed results of the runbook, click on the **Output** tile.
8. In the **Output** blade, you should see it has successfully authenticated and returned a list of all resources available in the resource group.
9. Close the **Output** blade to return to the **Job Summary** blade.
10. Close the **Job Summary** and the corresponding **AzureAutomationTutorialScript** runbook blade.

## Verify Classic Run As authentication
Next we will perform a small test to confirm you are able to successfully authenticate using the new Classic Run As account.     

1. In the Azure portal, open the Automation account created earlier.  
2. Click on the **Runbooks** tile to open the list of runbooks.
3. Select the **AzureClassicAutomationTutorialScript** runbook and then click **Start** to  start the runbook.  You will receive a prompt verifying you wish to start the runbook.
4. A [runbook job](automation-runbook-execution.md) is created, the Job blade is displayed, and the job status displayed in the **Job Summary** tile.  
5. The job status will start as *Queued* indicating that it is waiting for a runbook worker in the cloud to become available. It will then move to *Starting* when a worker claims the job, and then *Running* when the runbook actually starts running.  
6. When the runbook job completes, we should see a status of **Completed**.<br> ![Security Principal Runbook Test](media/automation-sec-configure-azure-runas-account/job-summary-automationclassictutorialscript.png)<br>
7. To see the detailed results of the runbook, click on the **Output** tile.
8. In the **Output** blade, you should see it has successfully authenticated and returned a list of all classic VM’s in the subscription.
9. Close the **Output** blade to return to the **Job Summary** blade.
10. Close the **Job Summary** and the corresponding **AzureClassicAutomationTutorialScript** runbook blade.

## Managing Azure Run As account
During the lifetime of your Automation account, you will need to renew the certificate before it expires or if you believe the account has been compromised, you can delete the Run As account and re-create it.  This section will provide steps on how to perform these operations.  

### Self-signed certificate renewal
The self-signed certificate created for the Azure Run As account can be renewed at anytime, up until it expires, which is one year from date of creation.  When you renew it, the old valid certificate will be retained in order to ensure that any runbooks queued up or actively running, which authenticate with the Run As account, will not be impacted.  The certificate will continue to exist until expiration.    

> [!NOTE]
> If you have configured your Automation Run As account to use a certificate issued by your enterprise certificate authority and you use this option, that certificate will be replaced by a self-signed certificate.  

1. In the Azure portal, open the Automation account.  
2. On the Automation account blade, in the account properties pane, select **Run As Accounts** under the section **Account Settings**.<br><br> ![Automation account properties pane](media/automation-sec-configure-azure-runas-account/automation-account-properties-pane.png)<br><br>
3. On the **Run As Accounts** properties blade, select either the Run As Account or Classic Run As account that you wish to renew the certificate for, and on the properties blade for the selected account, click **Renew certificate**.<br><br> ![Renew certificate for Run As account](media/automation-sec-configure-azure-runas-account/automation-account-renew-runas-certificate.png)<br><br> You will receive a prompt verifying you wish to proceed.  
4. While the certificate is being renewed, you can track the progress under **Notifications** from the menu.

### Delete Run As account
The following steps describe how to delete and re-create your Azure Run As or Classic Run As account.  When you perform this action the Automation account is retained.  After deleting the Run As or Classic Run As account, you can recreate it in the portal.  

1. In the Azure portal, open the Automation account.  
2. On the Automation account blade, in the account properties pane, select **Run As Accounts** under the section **Account Settings**.
3. On the **Run As Accounts** properties blade, select either the Run As Account or Classic Run As account that you wish to delete, and on the properties blade for the selected account, click **Delete**.<br><br> ![Delete Run As account](media/automation-sec-configure-azure-runas-account/automation-account-delete-runas.png)<br><br>  You will receive a prompt verifying you wish to proceed.
4. While the account is being deleted, you can track the progress under **Notifications** from the menu.  Once the deletion is complete, you can re-create it from the On the **Run As Accounts** properties blade and selecting the create option **Azure Run As Account**.<br><br> ![Recreate the Automation Run As account](media/automation-sec-configure-azure-runas-account/automation-account-create-runas.png)<br> 

### Misconfiguration
If any of the configuration items necessary for the Run As or Clasic Run As account to function properly are deleted or were not created properly during initial setup, such as:

* Certificate asset 
* Connection asset 
* Run As account has been removed from the contributor role
* Service principal or application in Azure AD

Automation will detect these changes and notify you with a status of **Incomplete** in the **Run As Accounts** properties blade for the account.<br><br> ![Incomplete Run As configuration status message](media/automation-sec-configure-azure-runas-account/automation-account-runas-incomplete-config.png)<br><br>When you select the Run As account, the following error will be presented in the properties pane of the account:<br><br> ![Incomplete Run As configuration warning message](media/automation-sec-configure-azure-runas-account/automation-account-runas-incomplete-config-msg.png).<br>  
If your Run As account is misconfigured, you can quickly resolve this by deleting and re-creating the Run As account.   

## Update an Automation Account using PowerShell
Here we provide you with the option to use PowerShell to update your existing Automation account if:

1. You created an Automation account, but declined to create the Run As account 
2. You already have an Automation account to manage Resource Manager resources and you want to update it to include the Run As account for runbook authentication
4. You already have an Automation account to manage classic resources and you want to update it to use the Classic Run As instead of creating a new account and migrating your runbooks and assets to it   
5. You want to create an Azure Run As and Classic Run As account using a certificate issued by your enterprise CA

This script has the following prerequisites:

1. This script supports running only on Windows 10 and Windows Server 2016 with Azure Resource Manager modules 2.01 and higher installed.  It is not supported on earlier versions of Windows.  
2. Azure PowerShell 1.0 and higher. For information about this release and how to install it, see [How to install and configure Azure PowerShell](/powershell/azureps-cmdlets-docs).
3. You have created an automation account.  This account will be referenced as the value for parameters –AutomationAccountName and -ApplicationDisplayName in the script below.

To get the values for *SubscriptionID*, *ResourceGroup*, and *AutomationAccountName*, which are required parameters for the scripts, in the Azure portal select your Automation account from the **Automation account** blade and select **All settings**.  From the **All settings** blade, under **Account Settings** select **Properties**.  In the **Properties** blade, you can note these values.<br><br> ![Automation Account properties](media/automation-sec-configure-azure-runas-account/automation-account-properties.png)  

### Create Run As Account PowerShell script
This PowerShell script includes support for the following configurations: 

* Create Azure Run As account using Self-Signed cert
* Create Azure Run As account and Azure Classic Run As account using Self-Signed cert
* Create Azure Run As account and Azure Classic Run As account using Enterprise cert
* Create Azure Run As account and Azure Classic Run As account using Self-Signed cert in Azure Government cloud

It will create the following depending on which configuration option you select:

* An Azure AD application that will be exported with either the self-signed or enterprise certificate public key, create a service principal account for this application in Azure AD, and assigned the Contributor role (you could change this to Owner or any other role) for this account in your current subscription.  For further information, please review the [Role-based access control in Azure Automation](automation-role-based-access-control.md) article.
* An Automation certificate asset in the specified automation account named **AzureRunAsCertificate**, which holds the certificate private key used by the Azure AD application.
* An Automation connection asset in the specified automation account named **AzureRunAsConnection**, which holds the applicationId, tenantId, subscriptionId, and certificate thumbprint.    

For Classic Run As account:

* An Automation certificate asset in the specified automation account named **AzureClassicRunAsCertificate**, which holds the certificate private key used by the management certificate.  
* An Automation connection asset in the specified automation account named **AzureClassicRunAsConnection**, which holds the subscription name, subscriptionId and certificate asset name.

If you select the option to create Classic Run As, after script execution you will need to upload the public certificate (.cer format) into the management store for the subscription the Automation account was created in. The steps below will walk you through the process of executing the script and uploading the certificate.    

1. Save the following script on your computer.  In this example, save it with the filename **New-RunAsAccount.ps1**.  
   
         #Requires -RunAsAdministrator
         Param (
        [Parameter(Mandatory=$true)]
        [String] $ResourceGroup,

        [Parameter(Mandatory=$true)]
        [String] $AutomationAccountName,

        [Parameter(Mandatory=$true)]
        [String] $ApplicationDisplayName,

        [Parameter(Mandatory=$true)]
        [String] $SubscriptionId,

        [Parameter(Mandatory=$true)]
        [Boolean] $CreateClassicRunAsAccount,

        [Parameter(Mandatory=$true)]
        [String] $SelfSignedCertPlainPassword,
 
        [Parameter(Mandatory=$false)]
        [String] $EnterpriseCertPathForRunAsAccount,

        [Parameter(Mandatory=$false)]
        [String] $EnterpriseCertPlainPasswordForRunAsAccount,

        [Parameter(Mandatory=$false)]
        [String] $EnterpriseCertPathForClassicRunAsAccount,

        [Parameter(Mandatory=$false)]
        [String] $EnterpriseCertPlainPasswordForClassicRunAsAccount,

        [Parameter(Mandatory=$false)]
        [ValidateSet("AzureCloud","AzureUSGovernment")]
        [string]$EnvironmentName="AzureCloud",

        [Parameter(Mandatory=$false)]
        [int] $SelfSignedCertNoOfMonthsUntilExpired = 12
        )

        function CreateSelfSignedCertificate([string] $keyVaultName, [string] $certificateName, [string] $selfSignedCertPlainPassword, 
                                      [string] $certPath, [string] $certPathCer, [string] $selfSignedCertNoOfMonthsUntilExpired ) {
        $Cert = New-SelfSignedCertificate -DnsName $certificateName -CertStoreLocation cert:\LocalMachine\My `
           -KeyExportPolicy Exportable -Provider "Microsoft Enhanced RSA and AES Cryptographic Provider" `
           -NotAfter (Get-Date).AddMonths($selfSignedCertNoOfMonthsUntilExpired)

        $CertPassword = ConvertTo-SecureString $selfSignedCertPlainPassword -AsPlainText -Force
        Export-PfxCertificate -Cert ("Cert:\localmachine\my\" + $Cert.Thumbprint) -FilePath $certPath -Password $CertPassword -Force | Write-Verbose
        Export-Certificate -Cert ("Cert:\localmachine\my\" + $Cert.Thumbprint) -FilePath $certPathCer -Type CERT | Write-Verbose 
        }

        function CreateServicePrincipal([System.Security.Cryptography.X509Certificates.X509Certificate2] $PfxCert, [string] $applicationDisplayName) {  
        $CurrentDate = Get-Date
        $keyValue = [System.Convert]::ToBase64String($PfxCert.GetRawCertData())
        $KeyId = (New-Guid).Guid 

        $KeyCredential = New-Object  Microsoft.Azure.Commands.Resources.Models.ActiveDirectory.PSADKeyCredential
        $KeyCredential.StartDate = $CurrentDate
        $KeyCredential.EndDate= [DateTime]$PfxCert.GetExpirationDateString()
        $KeyCredential.KeyId = $KeyId
        $KeyCredential.CertValue  = $keyValue

        # Use Key credentials and create AAD Application
        $Application = New-AzureRmADApplication -DisplayName $ApplicationDisplayName -HomePage ("http://" + $applicationDisplayName) -IdentifierUris ("http://" + $KeyId) -KeyCredentials $KeyCredential
        $ServicePrincipal = New-AzureRMADServicePrincipal -ApplicationId $Application.ApplicationId 
        $GetServicePrincipal = Get-AzureRmADServicePrincipal -ObjectId $ServicePrincipal.Id
   
        # Sleep here for a few seconds to allow the service principal application to become active (should only take a couple of seconds normally)
        Sleep -s 15
        $NewRole = New-AzureRMRoleAssignment -RoleDefinitionName Contributor -ServicePrincipalName $Application.ApplicationId -ErrorAction SilentlyContinue
        $Retries = 0;
        While ($NewRole -eq $null -and $Retries -le 6)
        {
           Sleep -s 10
           New-AzureRMRoleAssignment -RoleDefinitionName Contributor -ServicePrincipalName $Application.ApplicationId | Write-Verbose -ErrorAction SilentlyContinue
           $NewRole = Get-AzureRMRoleAssignment -ServicePrincipalName $Application.ApplicationId -ErrorAction SilentlyContinue
           $Retries++;
        }
           return $Application.ApplicationId.ToString();
        }

        function CreateAutomationCertificateAsset ([string] $resourceGroup, [string] $automationAccountName, [string] $certifcateAssetName,[string] $certPath, [string] $certPlainPassword, [Boolean] $Exportable) {
        $CertPassword = ConvertTo-SecureString $certPlainPassword -AsPlainText -Force   
        Remove-AzureRmAutomationCertificate -ResourceGroupName $resourceGroup -AutomationAccountName $automationAccountName -Name $certifcateAssetName -ErrorAction SilentlyContinue
        New-AzureRmAutomationCertificate -ResourceGroupName $resourceGroup -AutomationAccountName $automationAccountName -Path $certPath -Name $certifcateAssetName -Password $CertPassword -Exportable:$Exportable  | write-verbose
        }

        function CreateAutomationConnectionAsset ([string] $resourceGroup, [string] $automationAccountName, [string] $connectionAssetName, [string] $connectionTypeName, [System.Collections.Hashtable] $connectionFieldValues ) {
        Remove-AzureRmAutomationConnection -ResourceGroupName $resourceGroup -AutomationAccountName $automationAccountName -Name $connectionAssetName -Force -ErrorAction SilentlyContinue
        New-AzureRmAutomationConnection -ResourceGroupName $ResourceGroup -AutomationAccountName $automationAccountName -Name $connectionAssetName -ConnectionTypeName $connectionTypeName -ConnectionFieldValues $connectionFieldValues 
        }

        Import-Module AzureRM.Profile
        Import-Module AzureRM.Resources

        $AzureRMProfileVersion= (Get-Module AzureRM.Profile).Version
        if (!(($AzureRMProfileVersion.Major -ge 2 -and $AzureRMProfileVersion.Minor -ge 1) -or ($AzureRMProfileVersion.Major -gt 2)))
        {
           Write-Error -Message "Please install the latest Azure PowerShell and retry. Relevant doc url : https://docs.microsoft.com/powershell/azureps-cmdlets-docs/ "
           return
        }
 
        Login-AzureRmAccount -EnvironmentName $EnvironmentName
        $Subscription = Select-AzureRmSubscription -SubscriptionId $SubscriptionId

        # Create Run As Account using Service Principal
        $CertifcateAssetName = "AzureRunAsCertificate"
        $ConnectionAssetName = "AzureRunAsConnection"
        $ConnectionTypeName = "AzureServicePrincipal"
 
        if ($EnterpriseCertPathForRunAsAccount -and $EnterpriseCertPlainPasswordForRunAsAccount) {
        $PfxCertPathForRunAsAccount = $EnterpriseCertPathForRunAsAccount
        $PfxCertPlainPasswordForRunAsAccount = $EnterpriseCertPlainPasswordForRunAsAccount
        } else {
          $CertificateName = $AutomationAccountName+$CertifcateAssetName
          $PfxCertPathForRunAsAccount = Join-Path $env:TEMP ($CertificateName + ".pfx")
          $PfxCertPlainPasswordForRunAsAccount = $SelfSignedCertPlainPassword
          $CerCertPathForRunAsAccount = Join-Path $env:TEMP ($CertificateName + ".cer")
          CreateSelfSignedCertificate $KeyVaultName $CertificateName $PfxCertPlainPasswordForRunAsAccount $PfxCertPathForRunAsAccount $CerCertPathForRunAsAccount $SelfSignedCertNoOfMonthsUntilExpired 
        }

        # Create Service Principal
        $PfxCert = New-Object -TypeName System.Security.Cryptography.X509Certificates.X509Certificate2 -ArgumentList @($PfxCertPathForRunAsAccount, $PfxCertPlainPasswordForRunAsAccount)
        $ApplicationId=CreateServicePrincipal $PfxCert $ApplicationDisplayName

        # Create the automation certificate asset
        CreateAutomationCertificateAsset $ResourceGroup $AutomationAccountName $CertifcateAssetName $PfxCertPathForRunAsAccount $PfxCertPlainPasswordForRunAsAccount $true

        # Populate the ConnectionFieldValues
        $SubscriptionInfo = Get-AzureRmSubscription -SubscriptionId $SubscriptionId
        $TenantID = $SubscriptionInfo | Select TenantId -First 1
        $Thumbprint = $PfxCert.Thumbprint
        $ConnectionFieldValues = @{"ApplicationId" = $ApplicationId; "TenantId" = $TenantID.TenantId; "CertificateThumbprint" = $Thumbprint; "SubscriptionId" = $SubscriptionId} 

        # Create a Automation connection asset named AzureRunAsConnection in the Automation account. This connection uses the service principal.
        CreateAutomationConnectionAsset $ResourceGroup $AutomationAccountName $ConnectionAssetName $ConnectionTypeName $ConnectionFieldValues

        if ($CreateClassicRunAsAccount) {
            # Create Run As Account using Service Principal
            $ClassicRunAsAccountCertifcateAssetName = "AzureClassicRunAsCertificate"
            $ClassicRunAsAccountConnectionAssetName = "AzureClassicRunAsConnection"
            $ClassicRunAsAccountConnectionTypeName = "AzureClassicCertificate "
            $UploadMessage = "Please upload the .cer format of #CERT# to the Management store by following the steps below." + [Environment]::NewLine +
                    "Log in to the Microsoft Azure Management portal (https://manage.windowsazure.com) and select Settings -> Management Certificates." + [Environment]::NewLine +
                    "Then click Upload and upload the .cer format of #CERT#" 
 
             if ($EnterpriseCertPathForClassicRunAsAccount -and $EnterpriseCertPlainPasswordForClassicRunAsAccount ) {
             $PfxCertPathForClassicRunAsAccount = $EnterpriseCertPathForClassicRunAsAccount
             $PfxCertPlainPasswordForClassicRunAsAccount = $EnterpriseCertPlainPasswordForClassicRunAsAccount
             $UploadMessage = $UploadMessage.Replace("#CERT#", $PfxCertPathForClassicRunAsAccount)
        } else {
             $ClassicRunAsAccountCertificateName = $AutomationAccountName+$ClassicRunAsAccountCertifcateAssetName
             $PfxCertPathForClassicRunAsAccount = Join-Path $env:TEMP ($ClassicRunAsAccountCertificateName + ".pfx")
             $PfxCertPlainPasswordForClassicRunAsAccount = $SelfSignedCertPlainPassword
             $CerCertPathForClassicRunAsAccount = Join-Path $env:TEMP ($ClassicRunAsAccountCertificateName + ".cer")
             $UploadMessage = $UploadMessage.Replace("#CERT#", $CerCertPathForClassicRunAsAccount)
             CreateSelfSignedCertificate $KeyVaultName $ClassicRunAsAccountCertificateName $PfxCertPlainPasswordForClassicRunAsAccount $PfxCertPathForClassicRunAsAccount $CerCertPathForClassicRunAsAccount $SelfSignedCertNoOfMonthsUntilExpired 
        }

        # Create the automation certificate asset
        CreateAutomationCertificateAsset $ResourceGroup $AutomationAccountName $ClassicRunAsAccountCertifcateAssetName $PfxCertPathForClassicRunAsAccount $PfxCertPlainPasswordForClassicRunAsAccount $false

        # Populate the ConnectionFieldValues
        $SubscriptionName = $subscription.Subscription.SubscriptionName
        $ClassicRunAsAccountConnectionFieldValues = @{"SubscriptionName" = $SubscriptionName; "SubscriptionId" = $SubscriptionId; "CertificateAssetName" = $ClassicRunAsAccountCertifcateAssetName}

        # Create a Automation connection asset named AzureRunAsConnection in the Automation account. This connection uses the service principal.
        CreateAutomationConnectionAsset $ResourceGroup $AutomationAccountName $ClassicRunAsAccountConnectionAssetName $ClassicRunAsAccountConnectionTypeName $ClassicRunAsAccountConnectionFieldValues

        Write-Host -ForegroundColor red $UploadMessage
        }

2. On your computer, start **Windows PowerShell** from the **Start** screen with elevated user rights.
3. From the elevated PowerShell command-line shell, navigate to the folder which contains the script you created in Step 1 and execute the script setting the required parameter values based on the configuration you require.  

    **Create Azure Run As account using self-signed certificate**  
    `.\New-RunAsAccount.ps1 -ResourceGroup <ResourceGroupName> -AutomationAccountName <NameofAutomationAccount> -SubscriptionId <SubscriptionId> -ApplicationDisplayName <DisplayNameofAADApplication> -SelfSignedCertPlainPassword <StrongPassword> -CreateClassicRunAsAccount $false` 

    **Create Azure Run As account and Azure Classic Run As account using self-signed certificate**  
    `.\New-RunAsAccount.ps1 -ResourceGroup <ResourceGroupName> -AutomationAccountName <NameofAutomationAccount> -SubscriptionId <SubscriptionId> -ApplicationDisplayName <DisplayNameofAADApplication> -SelfSignedCertPlainPassword <StrongPassword> -CreateClassicRunAsAccount $true`

    **Create Azure Run As account and Azure Classic Run As account using enterprise certificate**  
    `.\New-RunAsAccount.ps1 -ResourceGroup <ResourceGroupName> -AutomationAccountName <NameofAutomationAccount> -SubscriptionId <SubscriptionId> -ApplicationDisplayName <DisplayNameofAADApplication>  -SelfSignedCertPlainPassword <StrongPassword> -CreateClassicRunAsAccount $true -EnterpriseCertPathForRunAsAccount <EnterpriseCertPfxPathForRunAsAccount> -EnterpriseCertPlainPasswordForRunAsAccount <StrongPassword> -EnterpriseCertPathForClassicRunAsAccount <EnterpriseCertPfxPathForClassicRunAsAccount> -EnterpriseCertPlainPasswordForClassicRunAsAccount <StrongPassword>`

    **Create Azure Run As account and Azure Classic Run As account using self-signed certificate in Azure Government cloud**  
    `.\New-RunAsAccount.ps1 -ResourceGroup <ResourceGroupName> -AutomationAccountName <NameofAutomationAccount> -SubscriptionId <SubscriptionId> -ApplicationDisplayName <DisplayNameofAADApplication> -SelfSignedCertPlainPassword <StrongPassword> -CreateClassicRunAsAccount $true  -EnvironmentName AzureUSGovernment`
 
    > [!NOTE]
    > You will be prompted to authenticate with Azure after you execute the script. You must log in with an account that is a member of the Subscription Admins role and co-admin of the subscription.
    > 
    > 

After the script completes successfully, if you created a Classic Run As account with self-signed public certificate (.cer format), the script will create and save it to the temporary files folder on your computer under the user profile used to execute the PowerShell session - *%USERPROFILE%\AppData\Local\Temp* or if you created a Classic Run As account with an enterprise public certificate (.cer format), you will need to use this certificate.  Follow the steps for [uploading a management API certificate](../azure-api-management-certs.md) to the Azure classic portal and then refer to the [sample code](#sample-code-to-authenticate-with-service-management-resources) to validate credential configuration with Service Management resources.  If you did not create a Classic Run As account, refer to the [sample code](#sample-code-to-authenticate-with-resource-manager-resources) below to authenticate with Resource Manager resources and validate credential configuration.

## Sample code to authenticate with Resource Manager resources
You can use the updated sample code below, taken from the **AzureAutomationTutorialScript** example runbook, to authenticate using the Run As account to manage Resource Manager resources with your runbooks.   

    $connectionName = "AzureRunAsConnection"
    $SubId = Get-AutomationVariable -Name 'SubscriptionId'
    try
    {
       # Get the connection "AzureRunAsConnection "
       $servicePrincipalConnection=Get-AutomationConnection -Name $connectionName         

       "Signing in to Azure..."
       Add-AzureRmAccount `
         -ServicePrincipal `
         -TenantId $servicePrincipalConnection.TenantId `
         -ApplicationId $servicePrincipalConnection.ApplicationId `
         -CertificateThumbprint $servicePrincipalConnection.CertificateThumbprint
       "Setting context to a specific subscription"     
       Set-AzureRmContext -SubscriptionId $SubId              
    }
    catch {
        if (!$servicePrincipalConnection)
        {
           $ErrorMessage = "Connection $connectionName not found."
           throw $ErrorMessage
         } else{
            Write-Error -Message $_.Exception
            throw $_.Exception
         }
    }


The script includes two additional lines of code to support referencing a subscription context so you can easily work between multiple subscriptions. A variable asset named SubscriptionId contains the ID of the subscription, and after the Add-AzureRmAccount cmdlet statement, the [Set-AzureRmContext cmdlet](https://msdn.microsoft.com/library/mt619263.aspx) is stated with the parameter set *-SubscriptionId*. If the variable name is too generic, you can revise the name of the variable to include a prefix or other naming convention to make it easier to identify for your purposes. Alternatively, you can use the parameter set -SubscriptionName instead of -SubscriptionId with a corresponding variable asset.    

Notice the cmdlet used for authenticating in the runbook - **Add-AzureRmAccount**, uses the *ServicePrincipalCertificate* parameter set.  It authenticates by using service principal certificate, not credentials.  

## Sample code to authenticate with Service Management resources
You can use the updated sample code below, taken from the **AzureClassicAutomationTutorialScript** example runbook, to authenticate using the Classic Run As account to manage classic resources with your runbooks.

    $ConnectionAssetName = "AzureClassicRunAsConnection"
    # Get the connection
    $connection = Get-AutomationConnection -Name $connectionAssetName        

    # Authenticate to Azure with certificate
    Write-Verbose "Get connection asset: $ConnectionAssetName" -Verbose
    $Conn = Get-AutomationConnection -Name $ConnectionAssetName
    if ($Conn -eq $null)
    {
       throw "Could not retrieve connection asset: $ConnectionAssetName. Assure that this asset exists in the Automation account."
    }

    $CertificateAssetName = $Conn.CertificateAssetName
    Write-Verbose "Getting the certificate: $CertificateAssetName" -Verbose
    $AzureCert = Get-AutomationCertificate -Name $CertificateAssetName
    if ($AzureCert -eq $null)
    {
       throw "Could not retrieve certificate asset: $CertificateAssetName. Assure that this asset exists in the Automation account."
    }

    Write-Verbose "Authenticating to Azure with certificate." -Verbose
    Set-AzureSubscription -SubscriptionName $Conn.SubscriptionName -SubscriptionId $Conn.SubscriptionID -Certificate $AzureCert
    Select-AzureSubscription -SubscriptionId $Conn.SubscriptionID

## Next steps
* For more information about Service Principals, refer to [Application Objects and Service Principal Objects](../active-directory/active-directory-application-objects.md).
* For more information about Role-based Access Control in Azure Automation, refer to [Role-based access control in Azure Automation](automation-role-based-access-control.md).
* For more information about certificates and Azure services, refer to [Certificates overview for Azure Cloud Services](../cloud-services/cloud-services-certs-create.md)

