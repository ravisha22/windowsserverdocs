---
title: Configure Windows 10 Client Always On VPN Connections
description: 
ms.prod: windows-server-threshold
ms.technology: networking
ms.topic: article
ms.assetid: d8cf3bae-45bf-4ffa-9205-290d555c59da
manager: brianlic
ms.author: pashort
author: shortpatti
ms.date: 3/4/2018
---

# STEP 7: Create the ProfileXML for Always On VPN Connections

>Applies To: Windows Server (Semi-Annual Channel), Windows Server 2016, Windows Server 2012 R2, Windows 10

ProfileXML is a URI node within the VPNv2 CSP. Rather than configuring each VPNv2 CSP node individually—such as triggers, route lists, and authentication protocols—use this node to configure a Windows 10 VPN client by delivering all the settings as a single XML block to a single CSP node. The ProfileXML schema matches the schema of the VPNv2 CSP nodes almost identically, but some terms are slightly different.

You use ProfileXML in all the delivery methods, including Windows PowerShell, System Center Configuration Manager, and Intune.

>[!NOTE] 
>Group Policy does not include administrative templates to configure the Windows 10 Remote Access Always On VPN client, however you can use logon scripts.

## Prerequisites

-   Make sure you review the [ProfileXML Overview](profilexml-overview.md) to have an understanding of the ProfileXML configuration files. The ProfileXML Overview section also includes the [MakeProfile.ps1 Full Script](profilexml-overview.md#full-script) section includes all of the code to generate two files: VPN_Profile.xml and VPN_Profile.ps1.

-   Ensure you have the host name or FQDN of the NPS from the server's certificate and the name of the CA that issued the certificate.

-   If you have not restarted the NPS server since configuring certificate autoenrollment, restart the NPS server to ensure you have a usable
    certificate enrolled on it.

## STEP 7.1: Configure the template VPN profile on a domain-joined client computer

Configure the template VPN profile on a domain-joined client computer. The type of user account you use, such as standard user or administrator, for this part of the process does not matter.

>[!NOTE] 
>There is no way to manually add any advanced properties of VPN, such as NRPT rules, Always On, Trusted network detection, etc. In the next step, you create a test VPN connection to verify that the VPN server is configured correctly, and to verify that you can establish a VPN connection to the server.

**Procedure:**

1.  Sign into a domain-joined client computer as a member of the **VPN Users** group.

2.  On the Start menu, type **VPN** and press Enter.

3.  Click **Add a VPN connection** and do the following:

    a.  In the **VPN Provider** list, select **Windows (built-in)**.

    b.  In **Connection Name**, type **Template**.

    c.  In **Server name or address**, enter the **external** FQDN of your VPN server, for example, **vpn.contoso.com**.

    d.  Click **Save**.

4.  In the right pane, under **Related Settings**, click **Change adapter options**.

5.  Right-click **Template**, and click **Properties** and do the following:

    a.  Click the **Security** tab, in **Type of VPN**, select **IKEv2**.

    d.  In **Data encryption**, click **Maximum strength encryption**.

    e.  Click the **Use Extensible Authentication Protocol (EAP)** check box and then select **Microsoft: Protected EAP (PEAP) (encryption enabled)** from the list.

    f.  Click **Properties** to open the Protected EAP Properties dialog box.

    g.  In the **Connect to these servers** box, type the name of the NPS server, for example, NPS01.<br><br>The server name you type must match the name in the certificate. You recovered this name earlier in this section. If the name does not match, the connection will fail, stating that “The connection was prevented because of a policy configured on your RAS/VPN server.”

    h.  Under **Trusted Root Certification Authorities**, select the root CA that issued the NPS server’s certificate, for example, contoso-CA.

    i.  In Notifications before connecting, select **Don’t ask user to authorize new servers or trusted CAs** from the list.

    j.  In Select Authentication Method, select **Smart Card or other certificate** from the list and click **Configure**.<br><br>The Smart Card or other Certificate Properties dialog opens.

    k.  Click **Use a certificate on this computer**.

    l. Click the **Connect to these servers** check box and enter the name of the NPS server.

    m. Under **Trusted Root Certification Authorities**, select the root CA that issued the NPS server’s certificate.

    n. Select the **Don’t prompt user to authorize new servers or trusted certification authorities** check box.

    o. Click **OK** to close the Smart Card or other Certificate Properties dialog box.

    p. Click **OK** to close the Protected EAP Properties dialog box.

6.  Click **OK** to close the Template Properties dialog box.

7.  Close the Network Connections window.

8.  In Settings, click **Template**, and clicking **Connect** to test the VPN.<br><br>Connect at least once before continuing; otherwise, the profile does not contain all the information necessary to connect to the VPN.

9.  Make sure that the template VPN connection to your VPN server is successful.<br><br>Doing so ensures that the EAP settings are correct.

## STEP 7.2: Prepare and create the ProfileXML configuration files

Before completing this section, make sure you have tested the VPN connection to ensure that the profile contains all the information required to connect to the VPN.

The MakeProfile.ps1 Windows PowerShell script creates two files on the desktop, both of which contain **EAPConfiguration** tags based on the template connection profile you created previously:

-   **VPN_Profile.xml.** This file contains the XML markup required to configure the ProfileXML node in the VPNv2 CSP. Use this file with OMA-DM–compatible MDM services, such as Intune.

-   **VPN_Profile.ps1.** This file is a Windows PowerShell script that you can run on client computers to configure the ProfileXML node in the VPNv2 CSP. You can also configure the CSP by deploying this script through SCCM. You cannot run this script in a Remote Desktop session, including a Hyper-V enhanced session.

>[!IMPORTANT] 
>The example commands below require Windows 10 Build 1607 or later.

**Procedure:**

1.  On the domain-joined client computer, open Windows PowerShell as an administrator.

2.  Copy the MakeProfile.ps1 Full Script from the [ProfileXML Overview](profilexml-overview.md#full-script) section and paste it into Windows PowerShell.

3.  Customize the following parameters:

    -   **\$Template**. The name of the template from which to retrieve the EAP configuration.

    -   **\$ProfileName**. Unique alpha numeric identifier for the profile. The
        profile name must not include a forward slash (/). If the profile name
        has a space or other non-alphanumeric character, it must be properly
        escaped according to the URL encoding standard.

    -   **\$Servers**. Public or routable IP address or DNS name for the VPN
        gateway. It can point to the external IP of a gateway or a virtual IP
        for a server farm. Examples, 208.147.66.130 or vpn.contoso.com.

    -   **\$DnsSuffix**. Specifies one or more comma-separated DNS suffixes. The
        first in the list is also used as the primary connection-specific DNS
        suffix for the VPN Interface. The entire list will also be added into
        the SuffixSearchList.

    -   **\$DomainName**. Used to indicate the namespace to which the policy
        applies. When a Name query is issued, the DNS client compares the name
        in the query to all the namespaces under DomainNameInformationList to
        find a match. This parameter can be one of the following types: FQDN and
        Suffix.

    -   **\$TrustedNetwork**. Comma-separated string to identify the trusted
        network. VPN will not connect automatically when the user is on their
        corporate wireless network where protected resources are directly
        accessible to the device.

    -   **\$DNSServers**. List of comma-separated DNS Server IP addresses to use for the namespace.<br><br>**Example values for parameters**
    ```
    $TemplateName = 'Template'
    $ProfileName = 'Contoso AlwaysOn VPN'
    $Servers = 'vpn.contoso.com'
    $DnsSuffix = 'corp.contoso.com'
    $DomainName = '.corp.contoso.com'
    $DNSServers = '10.10.0.2,10.10.0.3'
    $TrustedNetwork = 'corp.contoso.com'
    ```
4.  Run the **MakeProfile.ps1** script to generate VPN_Profile.xml and VPN_Profile.ps1 on the desktop.<br><br>**Example commands get EAP settings from the template profile**
    ```
    $Connection = Get-VpnConnection -Name $TemplateName
    if(!$Connection)
    {
    $Message = "Unable to get $TemplateName connection profile: $_"
    Write-Host "$Message"
    exit
    }
    $EAPSettings= $Connection.EapConfigXmlStream.InnerXml
    ```

    **Example of ProfileXML**
    ```
    $ProfileXML =
    '<VPNProfile>
      <DnsSuffix>' + $DnsSuffix + '</DnsSuffix>
      <NativeProfile>
    <Servers>' + $Servers + '</Servers>
    <NativeProtocolType>IKEv2</NativeProtocolType>
    <Authentication>
      <UserMethod>Eap</UserMethod>
      <Eap>
       <Configuration>
     '+ $EAPSettings + '
       </Configuration>
      </Eap>
    </Authentication>
    <RoutingPolicyType>SplitTunnel</RoutingPolicyType>
      </NativeProfile>
     <AlwaysOn>true</AlwaysOn>
     <RememberCredentials>true</RememberCredentials>
     <TrustedNetworkDetection>' + $TrustedNetwork + '</TrustedNetworkDetection>
      <DomainNameInformation>
    <DomainName>' + $DomainName + '</DomainName>
    <DnsServers>' + $DNSServers + '</DnsServers>
    </DomainNameInformation>
    </VPNProfile>'
    ```

    **Output VPN_Profile.xml for Intune**
    
    You can use the following example command to save the profile XML file.
    
    ```
    $ProfileXML | Out-File -FilePath ($env:USERPROFILE + '\desktop\VPN_Profile.xml')
    ```
    **Output VPN_Profile.ps1 for the desktop and System Center Configuration Manager**
    
    You can use this script on the Windows 10 desktop or in System Center Configuration Manager.
    
    ```
    $Script | Out-File -FilePath ($env:USERPROFILE + '\desktop\VPN_Profile.ps1')
    ```
5.  Define key VPN profile parameters.

    ```
    $Script = '$ProfileName = ''' + $ProfileName + '''
    $ProfileNameEscaped = $ProfileName -replace '' '', ''%20''
    ```

6.  Define VPN ProfileXML.

    ```
    $ProfileXML = ''' + $ProfileXML + '''
    ```

7.  Escape special characters in the profile.

    ```
    $ProfileXML = $ProfileXML -replace ''<'', ''&lt;''
    $ProfileXML = $ProfileXML -replace ''>'', ''&gt;''
    $ProfileXML = $ProfileXML -replace ''"'', ''&quot;''
    ```
8.  Define WMI-to-CSP Bridge properties.

    ```
    $nodeCSPURI = ''./Vendor/MSFT/VPNv2''
    $namespaceName = ''root\cimv2\mdm\dmmap''
    $className = ''MDM_VPNv2_01''
    ```
9.  Determine user SID for VPN profile.

```
try
{
$username = Gwmi -Class Win32_ComputerSystem | select username
$objuser = New-Object System.Security.Principal.NTAccount($username.username)
$sid = $objuser.Translate([System.Security.Principal.SecurityIdentifier])
$SidValue = $sid.Value
$Message = "User SID is $SidValue."
Write-Host "$Message"
}
catch [Exception]
{
$Message = "Unable to get user SID. User may be logged on over Remote Desktop: $_"
Write-Host "$Message"
exit
}
```

10.  Define WMI session.

```
$session = New-CimSession
$options = New-Object Microsoft.Management.Infrastructure.Options.CimOperationOptions
$options.SetCustomOption(''PolicyPlatformContext_PrincipalContext_Type'', ''PolicyPlatform_UserContext'', $false)
$options.SetCustomOption(''PolicyPlatformContext_PrincipalContext_Id'', "$SidValue", $false)
```

11.  Detect and delete previous VPN profile.

```PowerShell
try
{
    $deleteInstances = $session.EnumerateInstances($namespaceName, $className, $options)
    foreach ($deleteInstance in $deleteInstances)
    {
        $InstanceId = $deleteInstance.InstanceID
        if ("$InstanceId" -eq "$ProfileNameEscaped")
        {
            $session.DeleteInstance($namespaceName, $deleteInstance, $options)
            $Message = "Removed $ProfileName profile $InstanceId"
            Write-Host "$Message"
        } else {
            $Message = "Ignoring existing VPN profile $InstanceId"
            Write-Host "$Message"
        }
    }
}
catch [Exception]
{
    $Message = "Unable to remove existing outdated instance(s) of $ProfileName profile: $_"
    Write-Host "$Message"
    exit
}
```

12.  Create the VPN profile.<br>
```PowerShell
try
{
    $newInstance = New-Object Microsoft.Management.Infrastructure.CimInstance $className, $namespaceName
    $property = [Microsoft.Management.Infrastructure.CimProperty]::Create("ParentID", "$nodeCSPURI", ''String'', ''Key'')
    $newInstance.CimInstanceProperties.Add($property)
    $property = [Microsoft.Management.Infrastructure.CimProperty]::Create("InstanceID", "$ProfileNameEscaped", ''String'', ''Key'')
    $newInstance.CimInstanceProperties.Add($property)
    $property = [Microsoft.Management.Infrastructure.CimProperty]::Create("ProfileXML", "$ProfileXML", ''String'', ''Property'')
    $newInstance.CimInstanceProperties.Add($property)
    $session.CreateInstance($namespaceName, $newInstance, $options)
    $Message = "Created $ProfileName profile."


    Write-Host "$Message"
}
catch [Exception]
{
$Message = "Unable to create $ProfileName profile: $_"
Write-Host "$Message"
exit
}

$Message = "Script Complete"
Write-Host "$Message"'
```

13.  Save the VPN configuration file.
- **Intune**:

    ```
    $ProfileXML | Out-File -FilePath ($env:USERPROFILE + '\desktop\VPN_Profile.xml')
    ```
- **System Center Configuration Manager:**<br>
    ```
    $Script | Out-File -FilePath ($env:USERPROFILE + '\desktop\VPN_Profile.ps1')
    
    $Message = "Successfully created VPN_Profile.xml and VPN_Profile.ps1 on the desktop."
    Write-Host "$Message"
    ```

## Next steps

[STEP 8: Configure Windows 10 Client Always On VPN Connections](vpn-deploy-client-vpn-connections.md). After the Always On VPN infrastructure is ready, you create and publish the required certificates to the client. When the clients have received the certificates, you deploy the VPN_Profile.ps1 configuration script. Use Microsoft
System Center Configuration Manager or Microsoft Intune to monitor for successful VPN configuration deployments.