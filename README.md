 ### Create Home Lab running Active Directory using Oracle VirtualBox
![image](https://github.com/user-attachments/assets/1dd0cb65-cf50-48e5-ba9b-0bb9ef28835a)
This is my walkthrough and experience of the steps, broken down by tasks I created in Notion.

** 1.Download the required software: **
VirtualBox by Oracle — This was as simple as downloading the required installer, then running the installer on my local PC.
Windows Server 2019 ISO — This was downloaded from Microsoft, just an .iso image of Windows Server 2019.
Windows 10 ISO — This required downloading a Media Creation Tool, then creating an .iso image of Windows 10.

 2. Once VirtualBox was installed and the .iso images were downloaded, I launched VirtualBox. I created a new VM, named it DC, set the Memory size to 2048 MB in accordance with my local PC’s available RAM, left the disk size, file location, and size settings as default, then selected Create. Once the VM was created, I selected the 3 dots to access the settings. On the General pane of settings, under the Advanced tab, I set Shared Clipboard and Drag’n’Drop to Bidirectional to allow Copy/Paste and dragging of files between VM and local PC. On the System pane, under Processor tab, I made sure 1 CPU was selected for use in accordance with the available CPU cores on my local PC. Under the Network pane, for Adapter 1, I set “Attached to:” to NAT, then Adapter 2 I enabled and set “Attached to:” to Internal Network. I clicked Ok to complete the configuration settings of the VM.
![image](https://github.com/user-attachments/assets/b195617a-39cd-4e18-8edb-a4ef87441c39)

Network settings for the Windows Server 2019 VM
 3. Once the VM was configured, I started the VM from VirtualBox. Once it booted, I was prompted to select a disk or optical drive to start the virtual machine from. Using the File Explorer provided, I accessed the .iso file for Windows Server 2019 that was downloaded earlier, clicked Ok, then booted into Windows Server 2019 installation. I selected the Standard Evaluation (Desktop Experience) as the version of the OS to install, as the other options not Desktop Experience provide no visual GUI. After these settings were selected, I allowed Windows to reboot the VM several times to complete the installation.

### 4. With Server 2019 installed, I was now able to set it up and configure settings. I initially set the password for the built-in administrator account, then logged in as the built-in administrator. I selected the Network settings, then selected Change Adapter options. From here, I renamed the network connections to signify which one was which: _INTERNET_ for the public facing adapter, X_Internal_X for the internal network.
![image](https://github.com/user-attachments/assets/f6d7e206-efd6-4f44-91b1-a044fd67c375)


Static IP information set for the X_Internal_X network adapter
After the network adapters were named properly, I then set a static IP configuration on the X_Internal_X adapter to the following:

IP Address: 172.16.20.1
Subnet mask: 255.255.255.0
Default Gateway: intentionally left blank as the DC will serve as the gateway
DNS: 127.0.0.1 (loopback)

After that, I went to Control Panel > System > Advanced System Settings > then renamed the PC to the name I set in VirtualBox, “DC.” Once this was set, I restarted the VM.

5. I again logged into the VM, then from Server Manager, selected Add Roles and Features > followed the wizard and selected Active Directory Domain Services and followed the installation to completion.

Once the install completed, Server Manager had a yellow flag to begin post-deployment configuration by promoting this VM to a domain controller. On the Deployment Configuration page, I selected “Add a new forest” option, then entered a domain of “mydomain.com.” I set the DSRM password, then left all other settings default and finalized the installation, which prompted a reboot of the VM. After the VM rebooted, now the login prompt has the domain “mydomain.com” specified.

 6. I logged back into the VM, then immediately selected Administrative Tools from the Start menu. I then selected Active Directory Users and Computers, and created a new Organizational Unit named _ADMINS. It’s never a good practice to continue to use the built-in administrator account once Active Directory is setup, so in this new OU I created a new user account for myself, then added myself as a member of the Domain Admins group. Once my account was setup, I logged out of the built-in administrator account, then logged in using the new domain admin account.

Now it was time to setup RAS. From Server Manager, I selected Add Roles and Features > selected Remote Access, then completed the installation. Back in Server Manager, I selected Tools > Routing and Remote Access > right clicked DC > Configure and Enable Routing and Remote Access. Then under Configuration, I selected NAT > selected the network adapter we named _INTERNET_ and completed the configuration.
![image](https://github.com/user-attachments/assets/3b112c92-0252-41ce-a094-6beb29db1cd9)


RAS configuration for DC
 7. Now that RAS is setup, there should be one more thing left to do: setup DHCP. Also,tutorial provided a really awesomely powerful Powershell script to create new AD users automatically from a .txt file, so I will include that as well. From Server Manager, I went to Add Roles and Features > selected DHCP, then let the installation complete. I then selected Tools from Server Manager > DHCP. In the new window, I expanded dc.mydomain.com, then right clicked IPv4 > New Scope > added range of 172.16.0.100–200 and CIDR prefix length to 24 for a subnet mask of 255.255.255.0 > set no exclusions > left lease duration to 8 days since it doesn’t matter so much for a home lab, then went to the DHCP options. I set a Default Gateway of 172.16.0.1, then left the parent domain as mydomain.com, and activated the scope. I then right-clicked DHCP server, selected Authorize, then right-clicked again to Refresh. I now see a green circle and checkbox.
![image](https://github.com/user-attachments/assets/75343add-025e-411f-bbf9-c655267321d6)


DHCP settings for DC
From the VM, I then downloaded PS script to add additional users. It can be found at his GitHub here.

# ----- Edit these Variables for your own Use Case ----- #
$PASSWORD_FOR_USERS   = "Password1"
$USER_FIRST_LAST_LIST = Get-Content .\names.txt
# ------------------------------------------------------ #

$password = ConvertTo-SecureString $PASSWORD_FOR_USERS -AsPlainText -Force
New-ADOrganizationalUnit -Name _USERS -ProtectedFromAccidentalDeletion $false

foreach ($n in $USER_FIRST_LAST_LIST) {
    $first = $n.Split(" ")[0].ToLower()
    $last = $n.Split(" ")[1].ToLower()
    $username = "$($first.Substring(0,1))$($last)".ToLower()
    Write-Host "Creating user: $($username)" -BackgroundColor Black -ForegroundColor Cyan
    
    New-AdUser -AccountPassword $password 
               -GivenName $first 
               -Surname $last 
               -DisplayName $username 
               -Name $username 
               -EmployeeID $username 
               -PasswordNeverExpires $true 
               -Path "ou=_USERS,$(([ADSI]"").distinguishedName)" 
               -Enabled $true
}
I downloaded the .zip file contained the PS script, then extracted to the desktop. I accessed the .txt file containing 100’s of randomly generated names, then added my own name  to the top of the list, then saved and closed the .txt file. I launched Powershell ISE as administrator, then opened the PS script from the folder. Prior to running the script, I ran Set-ExecutionPolicy Unrestricted, then selected Yes to All to allow the script to run. I then had to change the directory in PS to the folder location of the script. From there, I was able to run the script successfully.
![image](https://github.com/user-attachments/assets/831099a4-1f4c-4053-b5ae-4548cd25cde0)


Powershell script to create users in AD provided by  Madakor. You can see in the script it also creates a new organizational unit in Active Directory for the users as well.

I then went to Active Directory to verify the users were created.

![image](https://github.com/user-attachments/assets/de69e701-6f75-4f95-bff1-ad5a993bf1a5)

As noted above, the script also created a new organization unit for the users named “_USERS.”
With the Server VM completed, now all that’s left to do is create a new VM for a Workstation, install Windows 10 on it, then configure it for usage within the newly created Active Directory Domain network we just created.

8. From VirtualBox, I created a new VM: ###

Name: Client1
Memory: 2048 MB
Left all other settings default and created VM > success.
Selected VM settings, then access Network pane, Adapter 1 tab > switched “Attached to: “ to Internal Network > success.
![image](https://github.com/user-attachments/assets/ae1a4978-9469-42be-b327-d4de4a11f4b2)


Network settings of the Windows 10 VM workstation.
Double clicked Client1 to start VM > prompted to add ISO > selected Windows 10 ISO > Windows 10 installs.

 9. Once Windows 10 installed, I began setting it up. I went through the steps to setup a local account instead of using a Microsoft account, then allowed Windows to complete setting up the local profile. Once my desktop loaded, I launched Control Panel from the Start menu, then went to System > Advanced System Settings > selected the Name tab, then selected Change to rename the PC and domain-join it at the same time > renamed the VM to CLIENT1, then joined mydomain.com by authenticating with the domain administrator credentials I created earlier. After finalizing, I was prompted to reboot the VM.

I then logged back into the VM with the newly created AD account we created from the Powershell script a step back to complete the lab.

![image](https://github.com/user-attachments/assets/05d7a5da-869e-46d5-a868-189d58b0e0bf)

The Windows 10 workstation showing it’s domain-joined successfully.

**Final Thoughts:

I feel like this was a really solid home lab that provided here. In my time in IT, it’s fairly difficult to find a really solid step-by-step walkthrough of all the steps required to complete a home lab project such as this that’s as clear and concise with the steps.
