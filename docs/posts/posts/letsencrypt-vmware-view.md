---
date: 2023-01-25
authors:
 - billyzoellers
categories:
    - LetsEncrypt
description: SSL certificate renewal using LetsEncrypt for VMware View VDI
slug: letsencrypt-vmware-view
---

# Let's Encrypt certificates for VMware Horizon

Recently, I needed to renew the SSL certificates on a VMware Horizon View deployment. I chose to make this the last year of paid SSL certifcates, and move the deployment over to Let's Encrypt. 

There are surely numerous approaches to do this with all code or just minimal code. In this case, the [Certify the Web](https://certifytheweb.com/)'s Windows application fit the bill for making future renewals and changes supportable by an IT staff that doesn't have much experience with scripting.

In this tutorial, Certify the Web is configured to handle the heavy lifting of Let's Encrypt's ACME certificate renewal process. PowerShell scripts are then used to deploy the newly minted certificates into VMware Horizon View and VMware's Unified Access Gateway (UAG) product.

<!-- more -->

## Certify the Web configuraiton
Install and configure Certify the Web on both the internal and external VMware Horizon Connection Server.

*[HCS]: Horizon Connection Server
*[UAG]: Unified Access Gateway

???+ warning "Perform these steps on all connection servers"
    
    The following configuration should be completed on both the internal and external VMware HCS. This process will result in each server having it's own unique certificate and private key. The UAG will share a key with the HCS it is paired with.

After the installation has completed, choose *New Certificate* and configure a new certificate using the parameters below:

### Certificate
1. Add your view domain `view.company.com` to the certificate. This will be the same on both VMware HCS.
2. Add a domain for this specific server. For example: `view-internal.company.com` for the internal HCS, or `view-external.company.com` for the external HCS. Mark this domain name as *primary*.
3. On the HCS paired with VMware UAG only, add a domain for the UAG. For example: `view-uag.company.com`.

???+ warning "Must use registered domains"

    You must use a public (registered) domain for this step even if the HCS or UAG do not have their own domain name in external DNS. Let's Encrypt will not issue certificates for IP addresses or `.local` domain names.

### Authorization
Select *Challenge Type* of `dns-01`. Make sure to configure the other settings on this page appropriately so that Certify the Web can create `_acme-challenge` records at your DNS provider of choice.

### Deployment
Choose *Deployment Mode* of `Certificate Store Only`.

### Tasks

#### All Horizon Connection Servers

1. Create a directory on the server for SSL automation tasks. For example `C:\ssl-automation`

2. Save this script in the new `ssl-automation` folder. Then create a *Run Powershell Script* task that will run the script:
    ``` powershell title="C:\ssl-automation\rename-cert.ps1" linenums="1"
    param($result)

    # Proceed only if Certify the Web reports success
    if ($result.IsSuccess) {
        # Remove any certificate named 'vdm' to avoid duplication
        Get-ChildItem -Path cert:\LocalMachine\My | Where {$_.FriendlyName.Equals("vdm")} | Remove-Item

        # Rename new Let's Encrypt certificate to 'vdm'
        $thumbprint = $result.ManagedItem.CertificateThumbprintHash
        $cert = Get-ChildItem -Path cert:\LocalMachine\My\$thumbprint
        $cert.FriendlyName ="vdm"
    }
    ```

2. Create a *Stop, Start or Restart a Service* task, choosing to restart the VMware Horizon View Connection Server (wsbroker) service. Extend the *Max. Wait Time (secs)* to 120 seconds.

#### Connection Server paired with UAG

???+ danger "External Connection Server only"

    The next two steps should be taken only on the VMware HCS paired with the UAG. This server will be responsible for renewal of the UAG's certificate, as well as it's own.

1. Create a *Deploy to Generic Server (multi-purpose)* task to export the key and full chain of the certificate. These files will be consumed by the PowerShell script in the next step.

    a. *Task Parameters* > *Output filepath for key*: `C:\ssl-automation\view.key`
    
    b. *Task Parameters* > *Output filepath for full chain*: `C:\ssl-automation\view.crt`

2. Save this script in the new `ssl-automation` folder. Then create a *Run Powershell Script* task that will run this script.

    You will need to set two variables in the `deploy-uag.ps1` file:

    `$uag_host`
        
    : This should be set to either the DNS resolvable hostname of your UAG server *or* it's IP address.

    `$uag_cred`

    : This should be set to the base64 encoding of `username:password` for your UAG server. You can create this value here: [base64encode.org](https://www.base64encode.org/)


    ``` powershell title="C:\ssl-automation\deloy-uag.ps1" linenums="1" hl_lines="3-6"
    param($result)

    ### UPDATE THESE VARIABLES
    $uag_host = "<hostname-of-UAG>"
    $uag_cred = "<api-key-of-UAG>"
    ### END OF VARIABLES

    # Ignore SSL errors for these API requests
    add-type @"
        using System.Net;
        using System.Security.Cryptography.X509Certificates;
        public class TrustAllCertsPolicy : ICertificatePolicy {
            public bool CheckValidationResult(
                ServicePoint srvPoint, X509Certificate certificate,
                WebRequest request, int certificateProblem) {
                return true;
            }
        }
    "@
    [System.Net.ServicePointManager]::CertificatePolicy = New-Object TrustAllCertsPolicy
    [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Ssl3, [Net.SecurityProtocolType]::Tls, [Net.SecurityProtocolType]::Tls11, [Net.SecurityProtocolType]::Tls12

    if ($result.IsSuccess) {
        # Transform PEM key and certificate into single text lines
        $key_oneline = (Get-Content -Path C:\ssl-automation\view.key) -join "\n"
        $cert_oneline = (Get-Content -Path C:\ssl-automation\view.crt) -join "\n"

        # Use API endpoints for both the 'internet' and 'admin' SSL certificate
        $endpoints = @(
            "https://$($uag_host):9443/rest/v1/config/certs/ssl",
            "https://$($uag_host):9443/rest/v1/config/certs/ssl/admin"
        )

        # Use REST to update the cert chain and key for both specified endpoints
        $data = '{"privateKeyPem":"' + $key_oneline + '","certChainPem":"' + $cert_oneline + '"}'
        $endpoints | ForEach-Object {
            Invoke-RestMethod `
            -Uri $_ `
            -Headers @{ "Authorization" = "Basic $uag_cred" } `
            -Method Put `
            -Body $data `
            -ContentType "application/json"
        }
    }
    ```

    ### Deploy the certificates
    Choose *Save* and then *Request Certificate* in Certify the Web. You can monitor the process status in the Certify the Web application.

