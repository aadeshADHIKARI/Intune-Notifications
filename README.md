#Requires -Version 5.0
#Requires -runasadministrator
<#
.SYNOPSIS
Sends a notification to a specified Teams Channel.
.DESCRIPTION
This script will send detailed information about an enrolled computer to a predefined Teams Channel.
.EXAMPLE
Run the script from a Win32app package or just as a PowerShell Configuration script *executed as system.
.NOTES
NAME: Intune-SlackNotification.ps1
VERSION: 1907d
PREREQ: Microsoft Teams (Duh!) 
    - Install the Incoming Webhook Connector to your Slack Channel.
    - Internet connectivity to the webhook and to psgallery
.COPYRIGHT
@michael_mardahl / https://www.iphase.dk
Licensed under the MIT license.
Please credit me if you fint this script useful and do some cool things with it.
Thanks go out to EvotecIT for creating the awesome PSTeams module! (https://github.com/EvotecIT)
#>

####################################################################################################
#
# Configuration Section (This is where you can edit stuff to fit your needs)
#
####################################################################################################

# The webhook URL you got from the Incoming Webhook Connector configuration guide
$WebhookURL = 'https://outlook.office.com/webhook/844208xrthrxtjxb10375320@ac3cfed8-c7xrtjxrtjd3a6e2b7/IncomingWebhook/58b1d853f3xjrxjrtjrxfe7c3c265b9/ff7xjrjxtrjtrjxt8b2885210'

# Enable testing mode (will post to the channel on every execution of this script if set to $true (default is $false))
$enableTesting = $false

##### Begin custom logic #####

    # Put any custom logic here, that you need to generate output for the Buttons or Facts in the notification.
    # I have added som example code as an inspiration

    # Get logged in user
    $currentUser = Get-WMIObject -class Win32_ComputerSystem | select -ExpandProperty username

    # Get teamviewer ID (if available)
    try { $teamviewerID = Get-ItemProperty -Path HKLM:\SOFTWARE\WOW6432Node\TeamViewer -ErrorAction Stop | select -ExpandProperty ClientID } catch { $teamviewerID = "N/A" }

    # Get OS install date
    $myinstallDate = ([WMI]'').ConvertToDateTime((Get-WmiObject Win32_OperatingSystem).InstallDate).ToString()

    # Get a nice display friendly OS Name
    $OSinfo = Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion"
    $OSDisplayName = "$($OSinfo.ProductName) $($OSinfo.ReleaseID)"

##### End custom logic #####

##### Design section starts here, and will determine the look of the notification

    # The color of top border of the Teams notification card.
    # Find more cool color names here: https://encycolorpedia.com/1e90ff
    $Color = "DodgerBlue"

    # The first two lines of our Notification
    $messageTitle = "$($env:COMPUTERNAME) just enrolled!"
    $messageText = "But it might not be finished provisioning yet...."

    $activityTitle = "Intune says... "
    $activitySubtitle = "I am collecting data about this device."
    $activityText = "You might find the following facts interesting!"
    $activityImageLink = "https://static-s.aa-cdn.net/img/gp/20600001711818/unUtqpVgwh3J6h_C4wmb0_Zc4ZuESSFejC9eJ8APpa8qy7EV1ulb1x9NufuSuBwm8A=w300"

    # Tables containing facts and buttons you wish to show in the notification
    # Add a new line for each fact / button, labels MUST be unique!
    # Fact Example: "Fact Label"="Custom text"
    # Button Example: "Button Label"="https://custom.url"
    # Markup: You can add markup to the custom text by wrapping a word or some text with *'s
    # ***Italic and Bold*** - **Bold** - *Italic*
    # Links: You can add links within the text, like so: [scconfigmgr](https://www.scconfigmgr.com)

    $facts = [ordered]@{
        "Computername"      = "$($env:COMPUTERNAME)"
        "Operating System"  = "$OSDisplayName"
        "Install Date"      = "$myinstallDate"
    }

    $Buttons = [ordered]@{
        "Visit scconfigmgr.com"  = "https://www.scconfigmgr.com"
        "Visit iphase.dk"        = "https://www.iphase.dk"
    }

####################################################################################################
#
# Functions Section (This is a collection of usefull code we are using in the execution section)
#
####################################################################################################

function isNewInstall () {
    # Determine if testing is enabled, and skip checks.
    if ($enableTesting) {
        return $true
    }
    
    # Determine if this computer was enrolled more than a day ago
    $DMClientTime = Get-ItemPropertyValue HKLM:\SOFTWARE\Microsoft\Provisioning\Diagnostics\ConfigManager\DMClient -Name Time -ErrorAction SilentlyContinue | Get-Date
    $nowMinus24Hours = (get-date).AddHours(-24)

    # Placing a cookie file, so this script won't run again by mistake.
    $cookieFile = "$($env:windir)\Temp\intune_notification-cookie.txt"
    if (Test-Path -Path $cookieFile) {
        return $false
    } else {
        Write-Output "this file indicates that the 'intune-TeamsNotification.ps1' script has run on this computer" > $cookieFile 
    }

    if ($nowMinus24Hours -gt $DMClientTime) { 
        return $false 
    } else {
        return $true
    }
}

function Install-PSTeams () {
    # Installs or updates the required PSTeams module
    if (!(Get-Module PSTeams -ListAvailable)) { 
        try {
            Install-Module PSTeams -Force -ErrorAction Stop
        } catch {
            Throw "Failed to install the required PSTeams Module! Better check your self, before you wreck your self!"
        }        
    } else {
        Update-Module PSTeams
        Import-Module PSTeams
    }
}

function makeButtonsFromHashtable($Hashtable) {
    # Generating Buttons Code from Hashtable in config section
    foreach ($Button in $Hashtable.Keys) {
        New-TeamsButton -Name $Button -Link $Hashtable["$Button"]
    }
}

function makeFactsFromHashtable($Hashtable) {
    # Generating Facts Code from Hashtable in config section
    # Currently this does not support multi-line facts, instead you will have to add those kind of facts manually.
    foreach ($Fact in $Hashtable.Keys) {
        New-TeamsFact -Name $Fact -Value "$($Hashtable["$Fact"])"
    }
}

####################################################################################################
#
# Execution Section (This is where stuff actually get's run!)
#
####################################################################################################

# Determine if this computer was recently installed or not (we dont want to send a notification from all previously enrolled computers)
if ((isNewInstall) -eq $false) {
    Write-Output "This computer was enrolled more than a day ago, so we wont't send a notification to the Teams Channel."
    Exit 0
}

# Installing the PSTeams Module if unavailable
Install-PSTeams

# Building the notification design
$Section1 = New-TeamsSection `
    -ActivityTitle "$activityTitle" `
    -ActivitySubtitle $activitySubtitle `
    -ActivityImageLink $activityImageLink `
    -ActivityText $activityText `
    -Buttons (makeButtonsFromHashtable -Hashtable $Buttons) `
    -ActivityDetails $(makeFactsFromHashtable -Hashtable $Facts)

# Sending the notification to the channel
Send-SlackMessage `
    -URI $WebhookURL `
    -MessageTitle $messageTitle `
    -MessageText $messageText `
    -Color $Color `
    -Sections $Section1 `
    -Verbose

