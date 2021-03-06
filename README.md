# Macco365
An Automated security assessment of Microsoft 365 tenants 

## Objective

To gain fast visability into the security state of Microsoft 365 environments for emergency onboardings and audits.

## Setup

Macco365 requires the administrative PowerShell modules for Microsoft Online, Azure AD, Exchange administration, and Sharepoint administration. 

If you do not have these modules installed, you should be able to install them with the following commands in an administrative PowerShell session, or by following the instructions at the references below:

	Install-Module -Name MSOnline
	Install-Module -Name AzureAD
	Install-Module -Name ExchangeOnlineManagement
	Install-Module -Name Microsoft.Online.SharePoint.PowerShell

* [Install MSOnline PowerShell](https://docs.microsoft.com/en-us/powershell/azure/active-directory/install-msonlinev1?view=azureadps-1.0)
* [Install Azure AD PowerShell](https://docs.microsoft.com/en-us/powershell/module/azuread/?view=azureadps-2.0)
* [Install Exchange Online PowerShell](https://docs.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2?view=exchange-ps)
* [Install SharePoint](https://docs.microsoft.com/en-us/powershell/sharepoint/sharepoint-online/connect-sharepoint-online?view=sharepoint-ps)

Once the above are installed, download the Macco365 source code folder from Github using your browser or by using *git clone*.

As you will run Macco365 with administrative privileges, you should place it in a LOGICAL location and ENSURE! the contents of the folder are readable and writable only by the administrative user (don't be a dumbass). This is especially important if you intend to install Macco365 in a location where it will be executed frequently or used as part of an automated process.

## Usage

To run Macco365, open a PowerShell console with local administrator privileges and navigate to the folder you downloaded Macco365 to:

	cd Macco365

You will interact with Macco365 by executing the main script file, Macco365.ps1, from within the PowerShell command prompt. 

All Macco365 requires to inspect your M365 tenant is access via an M365 account with the correct permissions, so most of the command line parameters relate to the organization being assessed and the method of authentication.

Execution of Macco365 looks like this:

	.\Macco365.ps1 -OrgName <value> -OutPath <value> -Auth <MFA|CMDLINE|ALREADY_AUTHED>

For example, to log in by entering your credentials in a browser with MFA support:

	.\Macco365.ps1 -OrgName mycompany -OutPath ..\365_report -Auth MFA

Or, with credentials passed on the command line:

	.\Macco365.ps1 -OrgName mycompany -OutPath ..\365_report -Auth CMDLINE -Username "first.last@soulsuckingcompany.com" -Password "d@mngo0dpa$$word420"

To break down the parameters further:

* *OrgName* is the name of the core organization or "company" of your M365 instance, which will be inspected. 
	* If you do not know the organization name, you can navigate to the list of all Exchange domains in M365. The topmost domain should be named *domain_name*.onmicrosoft.com. In that example, *domain_name* is your organization name and should be used when executing Macco365. Or you can just look on the first page of the AAD and if you did not know that you should NOT be using this script!
* *OutPath* is the path to a folder where the report generated by Macco365 will be placed.
* *Auth* is a selector that should be one of the literal values "MFA", "CMDLINE", or "ALREADY_AUTHED". 
	* *Auth* controls how Macco365 will authenticate to all of the Office 365 services. 
	* *Auth MFA* will produce a graphical popup in which you can type your credentials and even enter an MFA code for MFA-enabled accounts. 
	* *Auth CMDLINE* indicates that you intend to use a non-MFA-enabled account and pass the username and password on the command line, which may be preferable for automation integration or other tasks where headless execution is desired.
		* If you use *auth CMDLINE*, make sure to also pass the *username* and *password* parameters so Macco365 can log into your account without producing a popup window, as depicted in the 2nd example above.. 
	* *Auth ALREADY_AUTHED* instructs Macco365 not to authenticate before scanning. This may be preferable if you are executing Macco365 from a PowerShell prompt where you already have valid sessions for all of the described services, such as one where you have already executed Macco365.

When you execute Macco365 with *-Auth MFA*, it may produce several graphical login prompts that you must sequentially log into. This is normal behavior as Exchange, SharePoint etc. have separate administration modules and each requires a different login session. If you simply log in the requested number of times, Macco365 should begin to execute. This is the opposite of fun and we're seeking a workaround, but needless to say we feel the results are worth the minute spent looking at MFA codes.

As Macco365 executes, it will steadily print status updates indicating which inspection task is running.

Macco365 may take some time to execute. This time scales with the size and complexity of the environment under test. For example, some inspection tasks involve scanning the account configuration of all users. This may occur near-instantly for an organization with 50 users, or could take entire minutes (I know right, so damn long!) for an organization with 10000. 

## Output

Macco365 creates the directory specified in the out_path parameter. This directory is the result of the entire Macco365 inspection. It contains three items of note:

* *Report.html*: graphical report that describes the M365 security issues identified by Macco365, lists M365 objects that are misconfigured, and provides remediation advice.
* *Various text files named [Inspector-Name]*: these are raw output from inspector modules and contain a list (one item per line) of misconfigured M365 objects that contain the described security flaw. For example, if a module Inspect-FictionalMFASettings were to detect all users who do not have MFA set up, the file "Inspect-FictionalMFASettings" in the report ZIP would contain one user per line who does not have MFA set up. This information is only dumped to a file in cases where more than 15 affected objects are discovered. If less than 15 affected objects are discovered, the objects are listed directly in the main HTML report body.
* *Report.zip*: zipped version of this entire directory, for convenient distribution of the results in cases where some inspector modules generated a large amount of findings.

## Necessary Privileges

Macco365 can't run properly unless the M365 account you authenticate with has appropriate privileges. Macco365 requires, at minimum, the following:

* Global Reader
* Security Reader
* SharePoint Admin
* An Exchange role with View-Only access to everything

An extremely permissive role such as Global Admin will also work, but this isn't an appropriate long-term solution (DUH) if you intend to use Macco365 regularly or as part of an automated process.

## Developing Inspector Modules

Macco365 is designed to be easy to expand, with the hope that it enables individuals and organizations to either utilize their own Macco365 modules internally, or publish those modules for the M365 community.

All of Macco365's inspector modules are stored in the .\inspectors folder. 

It is simple to create an inspector module. Inspectors have two files:

* *ModuleName.ps1*: the PowerShell source code of the inspector module. Should return a list of all M365 objects affected by a specific issue, represented as strings.
* *ModuleName.json*: metadata about the inspector itself. For example, the finding name, description, remediation information, and references.

The PowerShell and JSON file names must be identical for Macco365 to recognize that the two belong together. There are numerous examples in Macco365's built-in suite of modules, but we'll put an example here too.

Example .ps1 file, BypassingSafeAttachments.ps1:
```powershell
# Define a function that we will later invoke.
# Macco365's built-in modules all follow this pattern.
function Inspect-BypassingSafeAttachments {
	# Query some element of the M365 environment to inspect. Note that we did not have to authenticate to Exchange
	# to fetch these transport rules within this module; assume main Macco365 harness has logged us in already.
	$safe_attachment_bypass_rules = (Get-TransportRule | Where { $_.SetHeaderName -eq "X-MS-Exchange-Organization-SkipSafeAttachmentProcessing" }).Identity
	
	# If some of the parsed M365 objects were found to have the security flaw this module is inspecting for,
	# return a list of strings representing those objects. This is what will end up as the "Affected Objects"
	# field in the report.
	If ($safe_attachment_bypass_rules.Count -ne 0) {
		return $safe_attachment_bypass_rules
	}
	
	# If none of the parsed M365 objects were found to have the security flaw this module is inspecting for,
	# returning $null indicates to Macco365 that there were no findings for this module.
	return $null
}

# Return the results of invoking the inspector function.
return Inspect-BypassingSafeAttachments
```

Example .json file, BypassingSafeAttachments.json:
```
{
	"FindingName": "Do Not Bypass the Safe Attachments Filter",
	"Description": "In Exchange, it is possible to create mail transport rules that bypass the Safe Attachments detection capability. The rules listed above bypass the Safe Attachments capability. Consider revie1wing these rules, as bypassing the Safe Attachments capability even for a subset of senders could be considered insecure depending on the context or may be an indicator of compromise.",
	"Remediation": "Navigate to the Mail Flow -> Rules screen in the Exchange Admin Center. Look for the offending rules and begin the process of assessing who created them and whether they are necessary to the continued function of your organization. If they are not, remove the rules.",
	"AffectedObjects": "",
	"References": [
		{
			"Url": "https://docs.microsoft.com/en-us/exchange/security-and-compliance/mail-flow-rules/manage-mail-flow-rules",
			"Text": "Manage Mail Flow Rules in Exchange Online"
		},
		{
			"Url": "https://www.undocumented-features.com/2018/05/10/atp-safe-attachments-safe-links-and-anti-phishing-policies-or-all-the-policies-you-can-shake-a-stick-at/#Bypass_Safe_Attachments_Processing",
			"Text": "Undocumented Features: Safe Attachments, Safe Links, and Anti-Phishing Policies"
		}
	]
}
```

Once you drop these two files in the .\inspectors folder, they are considered part of Macco365's module inventory and will run the next time you execute Macco365.

You have just created the BypassingSafeAttachments Inspector module. That's all!

Macco365 will throw a pretty loud and ugly error (very very Chono like) if something in your module doesn't work or doesn't follow Macco365 conventions, so monitor the command line output.

## About Security

Macco365 is a script harness that runs other inspector script modules stored in the .\inspectors folder. As with any other script you may run with elevated privileges, you should observe certain security hygiene practices:

* No untrusted user should have write access to the Macco365 folder/files, as that user could then overwrite scripts or templates therein and induce you to run malicious code.
* No script module should be placed in .\inspectors unless you trust the source of that script module.
* No idiots should be running this for the love of whatever diety, please do not tell the recruitment intern or Myyra to go run this script.

## Latah

@Gyarbij

Meaning: Macco: Noun - Somone who minds other people's business for the purpose of gossip (Caribbean Dictionary *vc*)
