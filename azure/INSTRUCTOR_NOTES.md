# Instructor Notes - Azure Dev/Test Labs

## What is this thing?
se-training-lab is a virtual classroom environment where you can quickly create Windows 10 workstations with HashiCorp tools pre-installed and ready to use. Windows was chosen as the OS because it commands [75% of the market share](https://www.statista.com/statistics/218089/global-market-share-of-windows-7/) for desktop operating systems. Windows 10 will be familiar and easy to use for the greatest number of users. These cloud desktop workstations are easy to use and provide a consistent environment for the benefit of instructors and students. The workstation image is currently published in two regions, **centralus** and **uksouth**. See below for a list of what's installed. These workstations can be used with pre-built training curriculum such as https://github.com/hashicorp/se-terraform-vault-workshop or any other lab exercise that requires Terraform or Vault command line tools.

![HashiCorp Windows 10 Cloud Workstation](https://github.com/hashicorp/se-terraform-vault-workshop/blob/master/windows_workstation.png)

## Setting up the lab
If you have access to the HashiCorp Azure Demo environment, you can simply browse to the Azure Dev/Test labs section and click on *se-training-lab* or *emea-training-lab*. If you're a partner or external user, skip down to the "Building an Azure Windows 10 Workstation" section to create a reusable custom image. Once you've got your custom Windows 10 image the process is pretty simple:

1. Enter your Azure Dev/Test lab page on the Azure portal.
1. Click on the blue plus sign at the top of the page: `+Add`
1. Browse to the standard SE training workstation image. It's named `selabworkstation`. This is a shared Azure Image Gallery image that is published to both centralus and uksouth.
1. Give the virtual machine a name. Pick something simple and short, preferably without special characters.
1. For user name enter `hashicorp`
1. For the password you can set your own. Make sure it meets the complexity requirements for Windows 10.
1. Under more options, click *Change Size* and select the B2ms class of machine. This size has 8GB of RAM which is enough for Windows 10.
1. Back up at the top click on Advanced Settings.
1. Change the ip address type to *Public*.
1. Set a deletion date. This is the date when your lab machines will be destroyed. 
1. Set the number of instances to the number of participants in your workshop, plus a few extras just in case.
1. Hit the Submit button at the bottom.
1. Wait about 15-20 minutes. When your machines are done building you'll see a little notification icon in the upper right corner.
1. Distribute the public URLs, username, and password to your students.

## Building an Azure Windows 10 Workstation - Packer
NEW: We now have a Packer template for building the workstation image.  You can find it here:

https://github.com/hashicorp/se-training-workstation
https://circleci.com/gh/hashicorp/se-training-workstation

## Building an Azure Windows 10 Workstation - Manual
The manual process below will be preserved for posterity.

1. Spin up a standard Windows 10 instance from the marketplace *inside Azure Dev/Test labs*. You'll use this as your base image. It's important that you create your machine inside the lab where you want to snapshot it. Once you're able to log onto the machine, run the steps below.

2. Run this script.
```
Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

choco install cmder -y
choco install git -y
choco install nmap -y
choco install 7zip -y
choco install putty -y
choco install openssh -y
choco install winscp -y
choco install visualstudiocode -y
choco install googlechrome -y
choco install poshgit -y
choco install jq -y
choco install azure-cli -y

# Create a Desktop shortcut for Cmder
# Note: Set your default shell to Powershell the first time you run this.
$TargetFile = "C:\tools\cmder\Cmder.exe"
$ShortcutFile = "C:\Users\Public\Desktop\cmder.lnk"
$WScriptShell = New-Object -ComObject WScript.Shell
$Shortcut = $WScriptShell.CreateShortcut($ShortcutFile)
$Shortcut.TargetPath = $TargetFile
$Shortcut.Save()
```
3. Install vault and terraform in C:\windows\system32. That's right, just toss the binaries in there.
4. Generate a token on the CAM vault server scoped to the following policy. This needs to be in the `Sales/SE` namespace. The following will create a token valid for 90 days. So remember to rebuild your workstation images at least once a quarter.
```
vault token create -policy=se-workshop-creds -ttl 2160h
```
5. Bake the token and CAM Vault URL into *system* environment variables. These are used by the setup_azure.ps1 script on the user's desktop.
```
[Environment]::SetEnvironmentVariable("SETUP_VAULT_TOKEN", "YOURTOKENHERE", "Machine")
[Environment]::SetEnvironmentVariable("SETUP_VAULT_ADDR", "https://cam-vault.hashidemos.io:8200", "Machine")
```
6. Add a file called `setup_azure.ps1` inside of `C:\Users\Public\Public Desktop`. This will ensure that it shows up on your users desktop when you deploy new workstations. This script fetches Azure credentials that are good for eight hours.

```
# Fetch dynamic Azure credentials for the workshop.
# Uses https://cam-vault.hashidemos.io:8200 and the Sales/SE namespace

# Fix git line ending settings on Windows
Set-Content -Path 'C:\Users\hashicorp\.gitconfig' -Value "[core]`n        autocrlf = false"

$VAULT_TOKEN = $env:SETUP_VAULT_TOKEN
$VAULT_ADDR = $env:SETUP_VAULT_ADDR

Write-Host -ForegroundColor Magenta "Fetching dynamic Azure credentials from HashiCorp Vault..."

$CREDS=(Invoke-RestMethod -Headers @{"X-Vault-Token" = ${VAULT_TOKEN}; "X-Vault-Namespace" = "Sales/SE"} -Method GET -Uri ${VAULT_ADDR}/v1/azure/creds/se-training-workstation).data

# write-host $CREDS
$CLIENT_ID=$CREDS.client_id
$CLIENT_SECRET=$CREDS.client_secret

Do {
    Write-Host -ForegroundColor White "Waiting for Azure credentials to be ready..."
    Start-Sleep 3
} Until (az login --service-principal -u 91299f64-f951-4462-8e97-9efb1d215501 -p $CLIENT_SECRET --tenant $env:ARM_TENANT_ID --allow-no-subscription 2> $null)

Write-Host -ForegroundColor Yellow "Storing credentials as system environment variables..."

[Environment]::SetEnvironmentVariable("ARM_SUBSCRIPTION_ID", "14692f20-9428-451b-8298-102ed4e39c2a", "Machine")
[Environment]::SetEnvironmentVariable("ARM_TENANT_ID", "0e3e2e88-8caf-41ca-b4da-e3b33b6c52ec", "Machine")
[Environment]::SetEnvironmentVariable("ARM_CLIENT_ID", "${CLIENT_ID}", "Machine")
[Environment]::SetEnvironmentVariable("ARM_CLIENT_SECRET", "${CLIENT_SECRET}", "Machine")

Write-Host -ForegroundColor DarkGreen "Dynamic credentials are good for 8 hours. You may proceed with the workshop."

# This is just for fun
Get-Content -Path C:\Users\Public\banner.txt

Read-Host -Prompt "Press Enter to Continue..."
```

7.  Run this to sysprep and "Generalize" the machine:

```
cd C:\windows\system32\sysprep
.\sysprep.exe /generalize
```

8.  Click the 'generalize' box and set the pulldown to "shutdown". Wait and give it a good ten minutes to fully shutdown.
9.  After the machine has been shut down, you can browse to it in the portal click it and create an image from it. Name it hc-training-workstation-DATE.  Example:  `hc-training-workstation-2019-05-12`
10. Use the image to spin up your workstations.
