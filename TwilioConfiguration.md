
<link id="linkstyle" rel='stylesheet' href='style.css'/>


OpCon - Phone Call Notifications via Twilio
===========

This solution will allow you to send phone call notifications in place of or in addition to OpCon's normal Notification Manager notifications.

## Twilio Configuration Values <a name="TwilioConfiguration"></a>

After creating a Twilio account, you can log into your Twilio console.  The Account SID and Authorization token are shown on the home page.

![Twilio Account](/img/TwilioAccount.png)

Record the Account SID and Auth Token. They will both be used by the OpCon Job initiating the call.

Next, click on the number sign (#) to see the Authorized phone number(s):

![Twilio Menu](/img/TwilioAccount2.png)

Record this telephone number.  It is the authorized telephone number for this account and is needed for the SMATwilioConnector configuration. 

## Twilio Powershell Script <a name="TwilioScript"></a>

```
<#
InitiateTwilioCall.ps1

Version History:

  2018/05/10: Initial creation with basic functionality and 
  logging.                                                   
                                                             
#>

param(
[string]$TwilioAccountSID,
[string]$TwilioAuthToken,
[string]$TwilioNumber,
[string]$PhoneNumber,
[string]$Message
)

try
{
    Add-Type -AssemblyName System.Web

    Write-Host "Target Phone Number: $PhoneNumber`n"
    Write-Host "Voice Message: $Message`n"

    $Message = [System.Web.HttpUtility]::UrlEncode($Message)

    $TwiMLUrl = "http://twimlets.com/message?Message%5b0%5d=$Message"

    Write-Host "TwiML URL: $TwimlUrl`n"

    # Twilio API endpoint and POST params
    $url = "https://api.twilio.com/2010-04-01/Accounts/$TwilioAccountSID/Calls.json"
    $params = @{ To = "+$PhoneNumber"; From = "+$TwilioNumber"; Url = $TwiMLUrl }

    # Create a credential object for HTTP basic auth
    $p = $TwilioAuthToken | ConvertTo-SecureString -asPlainText -Force
    $credential = New-Object System.Management.Automation.PSCredential($TwilioAccountSID, $p)

    # Make API request, selecting JSON properties from response
    $result = Invoke-WebRequest $url -Method Post -Credential $credential -Body $params -UseBasicParsing | ConvertFrom-Json

    Write-Host "Response: `n"
    $result | Format-List

}
catch
{
    $_.Exception | Format-List
    Exit 1
}
```

## Command Line Parameters <a name="CommandLine"></a>
There are five command line parameters requires for this script.
* -TwilioAccountSID
	* This is the Twilio Account SID found in the Twilio console on the home page.
* -TwilioAuthToken
	* This is the Authentication Tocken found in the Twilio console on the home page.
* -TwilioNumber
	* This is the sending phone number. This is the number that is tied to your Twilio Account.
* -PhoneNumber
	* This is the recieving phone number. A Global Property is suggested making it simple to update who is on call.
* -Message
	* This is the message that will be delieverd during the phone call.

## OpCon Job Setup <a name="JobSetup"></a>
These notifications will managed by an OpCon Schedule with on-demand multi-instance Jobs which will trigger the phone calls. Notification Manager will be setup to "Send OpCon/xps Events" adding the on-demand Jobs to the Schedule.

#### OnCall Alerts Schedule Configuration <a name="OnCallJobs"></a>
An OpCon Schedule will be built to manage the phone alerts. There will be at minimum two Jobs in this Schedule (additional configuration is required to set up escalating calls).
* Schedule Name "OnCall Alerts"
	* Start Time 00:00
	* Mark all days in the "Workays per Week"
	* Uncheck Use Master Holiday
	* Auto Build 0 days in advance for 2 days.
	* Auto Delete - use your company's standards.
* Job Name "Keep Schedule Open"
	* Null Job Type
	* Frequency - select a Frequency which allows the Job to run every day of the year.
	* Start Offset - 24:00
	* This Job make sure the Schedule is built every day and the Schedule is open allowing on-demand Jobs to be added.
* Job Name "Call Level One"
	* Windows Job Type
	* Check the Allow Multi-Instance checkbox
	* Select a Primary Machine which supported Embedded Scripts
	* Select the desired User Account
	* Select the appropriate embedded script
	* Select the PowerShell runner
	* Enter the following Arguments:
```
-TwilioAccountSID [[TwilioAccountSID]] -TwilioAuthToken [[TwilioAuthToken]] -TwilioNumber "[[TwilioNumber]]" -PhoneNumber "[[OnCallPrimary]]" -Message "Remember, OpCon can do anything if you put your mind to it."
```
*
	* Frequency - an OnRequest frequency which will only be built when called by an event
	* Run Intervale (these settings should be customized based on your preference)
		* Minutes from Start to Start 15
		* Number of Runs 5

