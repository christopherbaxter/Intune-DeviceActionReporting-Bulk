# Intune-DeviceActionReporting-Bulk

If you are migrating to Intune Bitlocker management, with Bitlocker Recovery Keys escrowed to AzureAD, this script will allow you to report on the results of the Bitlocker Key Rotation request. 

The reason this script exists is that (as of 15/02/2022), there is no other way to report on the result of the Bitlocker Rotation requests in bulk.

# Before you run this script!

Before you run this script, you will need to run the RotateBitlockerKeys-Parallel-RAW.ps1 file first, then leave the environment for a few days.

The script is detailed [Here](https://github.com/christopherbaxter/Intune-BitlockerKeyRotation-Bulk)

## What is needed for this script to function?

You will need a Service Principal in AzureAD with sufficient rights. I have a Service Principal that I use for multiple processes, I would not advise copying my permissions. I suggest following the guide from <https://msendpointmgr.com/2021/01/18/get-intune-managed-devices-without-an-escrowed-bitlocker-recovery-key-using-powershell/>. My permissions are set as in the image below. Please do not copy my permissions, this Service Principal is used for numerous tasks. I really should correct this, unfortunately, time has not been on my side, so I just work with what work for now. 

![](https://github.com/christopherbaxter/StaleComputerAccounts/blob/main/Images/ServicePrincipal%20-%20API%20Permissions.jpg)

I also elevate my AzureAD account to 'Intune Administrator', 'Cloud Device Administrator' and 'Security Reader'. These permissions also feel more than needed. Understand that I work in a very large environment, that is very fast paced, so I elevate these as I need them for other tasks as well.

You will need to make sure that you have the following PowerShell modules installed. There is a lot to consider with these modules as some cannot run with others. This was a bit of a learning curve. 

ActiveDirectory\
AzureAD\
ImportExcel\
JoinModule\
MSAL.PS\
PSReadline (May not be needed, not tested without this)

Ultimately, I built a VM on-prem in one of our data centres to run this script, including others. My machine has 4 procs and 16Gb RAM, the reason for an on-prem VM is because most of our workforce is working from home (me included), and running this script is a little slow through the VPN. Our ExpressRoute also makes this data collection significantly more efficient. In a small environment, you will not need this.

# Disclaimer

Ok, so my code may not be very pretty, or efficient in terms of coding. I have only been scripting with PowerShell since September 2020, have had very little (if any), formal PowerShell training and have no previous scripting experience to speak of, apart from the '1 liners' that AD engineers normally create, so please, go easy. I have found that I LOVE PowerShell and finding strange solutions like this have become a passion for me.

## Christopher, enough ramble, How does this thing work?

This script will extract all IntuneDeviceIDs from the MS Graph API. Once extracted, the script splits the IntuneDeviceID array into 30 smaller arrays, then will invoke a 'get' command to extract the Device Action log data from the MS Graph API.

### Parameters and Functions

The first section is where we supply the TenantID (Of the AzureAD tenant) and the ClientID of the Service Principal you have created. If you populate these (hard code), then the script will not ask for these and will immediately go to the Authentication process.

![](https://github.com/christopherbaxter/Intune-DeviceActionReporting-Bulk/blob/12c228e584a7562bf1e016a21b2f63f3c48e9cbd/Images/01-ParametersandFunctions.jpg)

The functions needed by the script are included in the script. I have modified the 'Invoke-MSGraphOperation' function significantly. I was running into issues with the token and renewing it. I also noted some of the errors went away with a retry or 2, so I built this into the function. Sorry @JankeSkanke @NickolajA for hacking at your work. :-)

### The Variables

The variable section also has a section to use the system proxy. I was having trouble with the proxy, intermittently. Adding these lines solved the problem

![](https://github.com/christopherbaxter/Intune-DeviceActionReporting-Bulk/blob/12c228e584a7562bf1e016a21b2f63f3c48e9cbd/Images/02-Variables.jpg)

### The initial Authentication and Token Acquisition

Ok, so now the 'fun' starts.

The authentication and token acquisition will allow for auth with MFA. You will notice in the script that I have these commands running a few times in the script. This allows for token renewal without requiring MFA again. I also ran into some strange issues with different MS Graph API resources, where a token used for one resource, could not be used on the next resource, this corrects this issue, no idea why, never dug too deep into it because I needed it to work, not be pretty. :-)

![](https://github.com/christopherbaxter/Intune-DeviceActionReporting-Bulk/blob/12c228e584a7562bf1e016a21b2f63f3c48e9cbd/Images/03-GenerateAuthTokenandAuthHeader.jpg)

### Intune Device Data Extraction

This section also requires an authentication process and will allow for MFA. The reason why I added this in here is that the script takes a long time to run in my environment, and so, if I perform this extraction first, without the initial auth\token process, the script will complete this process, then sit waiting for auth and MFA, and in essence, not run. Same if this was moved to after the MS Graph extractions. 

Having the 'authy' bits in this order, the script will ask for auth and MFA for MS Graph, then auth and MFA for AzureAD, one after the other with no delay, allowing the script to run without manual intervention. 

    $DevActionDevList = [System.Collections.ArrayList]@(Invoke-MSGraphOperation -Get -APIVersion "Beta" -Resource "deviceManagement/managedDevices?`$filter=operatingSystem eq 'Windows'" -Headers $AuthenticationHeader -Verbose | Where-Object { $_.azureADDeviceId -ne "00000000-0000-0000-0000-000000000000" } | Select-Object id | Sort-Object id)    

I extract the data into an ArrayList. This was needed for a previous 'join' function, I left it like this because I noted strange errors in other scripts. I never had the time to validate that this is the case here, so I simply left it in place. At some point, I would like to test other array types and test processing time between them, not now, this works exactly as needed.

![](https://github.com/christopherbaxter/Intune-DeviceActionReporting-Bulk/blob/12c228e584a7562bf1e016a21b2f63f3c48e9cbd/Images/04-ExtractIntuneDeviceIDs.jpg)

### Splitting the IntuneDeviceID array.

Now things get interesting, splitting the array into multiple equal parts. This was done as I was having trouble with the tokens expiring while the script was running, even with this process being managed in the function called "Invoke-MSGraphOperation", I just used the "Invoke-RestMethod" in the script.

![](https://github.com/christopherbaxter/Intune-DeviceActionReporting-Bulk/blob/12c228e584a7562bf1e016a21b2f63f3c48e9cbd/Images/05-SplitIntuneDeviceIDsArray.jpg)

### The Script block.

I was struggling with failures, either due to the authentication token expiring, or timeouts. I created the process to retry the command again. Keep in mind that i am running 30 simultaneous threads, and it is probable that I am overloading something along the way, hence the numerous retries.

![](https://github.com/christopherbaxter/Intune-DeviceActionReporting-Bulk/blob/12c228e584a7562bf1e016a21b2f63f3c48e9cbd/Images/06-ScriptblockforParallelProcessing.jpg)

### The Foreach Loop.

This is the key to the ability to run the commands in parallel. I made use of a module called PoshRSJob. This allowed me to run simultaneous web requests.

The foreach loop is setup to collect the full output from the MS Graph API.

![](https://github.com/christopherbaxter/Intune-DeviceActionReporting-Bulk/blob/12c228e584a7562bf1e016a21b2f63f3c48e9cbd/Images/07-ForeachLoop.jpg)

### Extracting the failures for the retry process.

The code for extracting the failures, works by 'blending' the extractions with the complete list of IntuneDeviceIDs, then extracting the devices without a null value in 'aadRegistered' or 'autopilotEnrolled' fields, and creating an array with these device IDs. Then splitting the array again like above.

    $FailedExtractList = [System.Collections.ArrayList]@($ExtractionCheck | 
    Where-Object { ($_.aadRegistered -like $null) -or ($_.autopilotEnrolled -like $null) } | 
    Select-Object id )

![](https://github.com/christopherbaxter/Intune-DeviceActionReporting-Bulk/blob/12c228e584a7562bf1e016a21b2f63f3c48e9cbd/Images/08-ProcesstheExtractandSelectFailed.jpg)

I then split the array again (same as above, only into 10 parts now)

![](https://github.com/christopherbaxter/Intune-DeviceActionReporting-Bulk/blob/12c228e584a7562bf1e016a21b2f63f3c48e9cbd/Images/09-SplitFailedArray.jpg)

### Retry the extract for the failures

There is not much more to see here, only that I again run the extract using the PoshRSJob module.

![](https://github.com/christopherbaxter/Intune-DeviceActionReporting-Bulk/blob/12c228e584a7562bf1e016a21b2f63f3c48e9cbd/Images/10-ForeachLoopToRetryFailedExtracts.jpg)

### The extracted data is 'blended' then exported

The code here is what performs the required transformation on the data in preparation for the Big reporting script. This data trnasformation looks like this (WARNING, ITS BIG):

    $DevActionArray = [System.Collections.ArrayList]@($ConsolidatedRawExtract | Select-Object 
    @{Name = "IntuneDeviceID"; Expression = { $_.id } }, 
    
    @{Name = "KeyRotationResult"; Expression = { if ($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object actionState ) { ($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object actionState).actionState } else { "NoRotationRequest" } } }, 
    
    @{Name = "KeyRotationRequestDate"; Expression = { if ($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object startDateTime ) { (Get-Date -Date ($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object startDateTime).startDateTime -Format "yyyy/MM/dd HH:mm") } else { "NoRotationRequest" } } }, 
    
    @{Name = "KeyRotationDate"; Expression = { if ($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object lastUpdatedDateTime ) { (Get-Date -Date ($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object lastUpdatedDateTime).lastUpdatedDateTime -Format "yyyy/MM/dd HH:mm") } else { "NoRotationRequest" } } }, 
    
    @{Name = "KeyRotationError"; Expression = { if (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorcode -like $null) { "NoRotationRequest" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2147023728") { "0x80070490" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272166") { "0x803100DA" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2147467259") { "0x80004005" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272310") { "0x8031004A" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2147024809") { "0x80070057" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272159") { "0x803100E1" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272165") { "0x803100DB" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272295") { "0x80310059" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272384") { "0x80310000" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272376") { "0x80310008" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2147418113") { "0x8000FFFF" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272366") { "0x80310012" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272339") { "0x8031002D" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2146893783") { "0x80090029" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "0") { "0" } else { ($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode } } }, 
    
    @{Name = "KeyRotationErrorDescription"; Expression = { if (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object actionState).actionState -like "Done" ) { "None" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2147023728") { "Element not Found" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272166") { "BitLocker recovery password rotation cannot be performed because backup policy for BitLocker recovery information is not set to required for the OS drive." }elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2147467259") { "General Failure." }elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272310") { "BitLocker Drive Encryption cannot be used because critical BitLocker system files are missing or corrupted. Use Windows Startup Repair to restore these files to your computer." }elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2147024809") { "One or more arguments are invalid - The parameter is incorrect." }elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272159") { "BitLocker recovery key backup endpoint is busy and cannot perform requested operation. Please retry after sometime." }elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272165") { "BitLocker recovery password rotation cannot be performed because backup policy for BitLocker recovery information is not set to required for fixed data drives." }elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272295") { "BitLocker Drive Encryption is already performing an operation on this drive. Please complete all operations before continuing." }elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272384") { "This drive is locked by BitLocker Drive Encryption. You must unlock this drive from Control Panel." }elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272376") { "BitLocker Drive Encryption is not enabled on this drive. Turn on BitLocker." }elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2147418113") { "Catastrophic failure" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272366") { "The drive cannot be encrypted because it contains system boot information. Create a separate partition for use as the system drive that contains the boot information and a second partition for use as the operating system drive and then encrypt the operating system drive." } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2144272339") { "The drive encryption algorithm and key cannot be set on a previously encrypted drive. To encrypt this drive with BitLocker Drive Encryption, remove the previous encryption and then turn on BitLocker." } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "-2146893783") { "The requested operation is not supported." } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "0") { if (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object actionState).actionState -match "Pending" ) { "Pending" } elseif (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object actionState).actionState -match "done" ) { "Key Rotation Successful." } elseif ({ (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object actionState).actionState -match "Failed") -and (($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object errorCode).errorCode -match "0") }) { ($_.deviceActionResults | Where-Object { $_.actionName -like "*BitLocker*" } | Select-Object actionState).actionState } } else { "Not Yet Determined." } } } | 
    Sort-Object IntuneDeviceID)

Basically, All this code does is takes the raw error code from the MS Graph API, converts it to the error code one would see in the Endpoint manager console, then give it a descriptive error description.

For example: Raw error is -2147023728, which means the error in the console is 0x80070490. The error means "Element not Found". I also added some code in to still output the raw error code if it is one not covered by the code above, allowing me to add it in if needed.

![](https://github.com/christopherbaxter/Intune-DeviceActionReporting-Bulk/blob/12c228e584a7562bf1e016a21b2f63f3c48e9cbd/Images/11-GenerateReportAndDumpExtract.jpg)