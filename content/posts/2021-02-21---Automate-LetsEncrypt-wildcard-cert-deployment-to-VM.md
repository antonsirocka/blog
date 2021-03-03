---
title: Automate Let's Encrypt wildcard certificate deployment to an Azure VM
date: "2021-02-21T14:48:32.169Z"
template: "post"
draft: false
slug: "automate-lets-encrypt-wildcard-certificate-deployment-to-an-azure-vm"
category: "Security"
tags:
  - "Azure"
  - "Virtual Machines"
  - "LetsEncrypt"
  - "Wildcard certificate"
description: "Automating TLS certificate issuing via Let's Encrypt is very straight-forward in new emerging orchestrators like Kubernetes. Achieving the same on a virtual machine running IIS is still very much in demand, but the process is not well documented and a little bit more difficult to get right."
socialImage: "../../media/letsencrypt-wildcard-automation.jpg"
---

Automating TLS certificate issuing via `Let's Encrypt` is very straight-forward in new emerging orchestrators like Kubernetes. Achieving the same on a virtual machine running IIS is still very much in demand, but the process is not well documented and a little bit more difficult to get right.

- [The agument for automating certificate issuing process](#the-agument-for-automating-certificate-issuing-process)
- [Let's Encrypt in Kubernetes vs Windows IIS](#lets-encrypt-in-kubernetes-vs-windows-iis)
- [The difference between wildcard and regular TLS certificates](#the-difference-between-wildcard-and-regular-tls-certificates)
- [Step 1 Automate wildcard certificate issuing using Azure functions and key vault](#step-1-automate-wildcard-certificate-issuing-using-azure-functions-and-key-vault)
- [Step 2 Link a VM with Azure KeyVault](#step-2-link-a-vm-with-azure-keyvault)
- [Step 3 Testing things out](#step-3-testing-things-out)
- [Step 4 Keeping bindings up to date with new certificates](#step-4-keeping-bindings-up-to-date-with-new-certificates)
- [Summary](#summary)

## The agument for automating certificate issuing process
Encrypting all traffic that travels over public internet in this day and age is a no-brainer. The introduction of nonprofit Certificate Authority (CA) like Let's Encrypt significantly streamlines the whole process. There are those who say trusting a nonprofit organisation with TLS encryption is a mistake, because there's less skin in the game on the CA behalf if they get hacked. I can see the argument from both angles, but strongly believe the old/manual way of issuing and installing certificates (having gone through the process myself very recently) is simply not fit for purpose in modern age.

## Let's Encrypt in Kubernetes vs Windows IIS
In Kubernetes world, automating certificate issuing is almost stupid-easy: Install cert-manager (via helm), deploy a cluster-issuer and add annotation to ingress (cert-manager.io/issuer: "letsencrypt"). Job done, you never have to worry about renewing a certificate again.

With virtual machines and IIS, the process is more involved. Especially if you plan to use wildcard certificates, but once you set it up correctly, you can reap the same benefits of not having to worry about regular certificate renewal.

## The difference between wildcard and regular TLS certificates

A regular certificate is only valid for a specific sub-domain it was issued for. For instance, `myblog.myworld.com`. Let's Encrypt needs to complete `HTTP-01 challenge` in order to issue such a certificate. Put simply, the CA (Let's Encrypt) generates a token and then tries to access `http://<YOUR_DOMAIN>/.well-known/acme-challenge/<TOKEN>`. If it gets a 200OK response, it is a proof that you own this domain, because you were able to set up a service that responds to their requests at the URL they specified.

A wildcard certificate is able to encrypt/decrypt traffic for a specific domain, but also any sub-domain (`*.myworld.com`). As such, the HTTP-01 challenge is no longer sufficient, because it is impossible to generate and verify a token for every possible subdomain that can exist. Therefore, a `DNS-01` challenge has to performed. It involves creating a specific DNS record for your domain (again, Let's Encrypt will decide what that record should look like). If you're able to create this record, and Let's Encrypt is able to access it, then again this will serve as proof that you are the owner this domain and a wildcard certificate can be issued.

You can read more about Let's Encrypt challenge types [here](https://letsencrypt.org/docs/challenge-types/).

## Step 1 Automate wildcard certificate issuing using Azure functions and key vault

The first objective is to automate wildcard certificate issuing and place the certificate in a key vault. A virtual machine can then link up with the key vault securely using a VM extension and download a new certificate when it becomes available.

Fortunately, clever people already did most of the leg-work for us and there is no point in trying to re-invent the wheel. Have a look at [Key Vault Acmebot](https://github.com/shibayan/keyvault-acmebot). The process is well documented. If you manage your DNS and Key Vault in Azure, it will take care of automatically creating the correct DNS record every 60 days and placing a new wildcard certificate in a key vault.

## Step 2 Link a VM with Azure KeyVault

To link up a VM with an Azure Key Vault, you can use an Azure VM extension. I'm a proponent of terraform, but you can equally just create the extension manually or using Azure deployment templates:

```yaml
resource "azurerm_virtual_machine_scale_set_extension" "keyvault_extension" {
  provider                     = azurerm.provideralias
  name                         = var.keyvault_extension_profile.name
  virtual_machine_scale_set_id = var.virtual_machine_scale_set_id
  publisher                    = var.keyvault_extension_profile.publisher
  type                         = var.keyvault_extension_profile.type
  type_handler_version         = var.keyvault_extension_profile.type_handler_version
  auto_upgrade_minor_version   = var.keyvault_extension_profile.auto_upgrade_minor_version
  settings = <<EOF
      {
        "secretsManagementSettings": {
          "pollingIntervalInS":       "${var.keyvault_extension_profile.polling_interval_in_s}",
          "certificateStoreName":     "${var.keyvault_extension_profile.certificate_store_name}",
          "certificateStoreLocation": "${var.keyvault_extension_profile.certificate_store_location}",
          "observedCertificates":     ${var.keyvault_extension_profile.observed_certificates}
        } 
      }
    EOF
}
```

You will need to provide a series of settings, so that the extension knows where to retrieve certificate from, how often and where to store it on the local machine. Here's an example of these settings:

```yaml
vmss_keyvault_extension_profile = {
  name                       = "KVVMExtensionForWindows"
  publisher                  = "Microsoft.Azure.KeyVault"
  type                       = "KeyVaultForWindows"
  type_handler_version       = "1.0" 
  auto_upgrade_minor_version = true
  polling_interval_in_s      = "3600"
  certificate_store_name     = "MY"
  link_on_renewal            = false
  certificate_store_location = "LocalMachine"
  require_initial_sync       = true
  observed_certificates  = "[\"https://kv-acmebot-xyzv.vault.azure.net/secrets/wildcard-certificate-name\"]"  
}
```

The key setting to pay attention to is `observed_certificates`:
- `kv-acmebot-xyzv` is the name of your key vault. Has to be globally unique.
- `wildcard-certificate-name` is the name of the certificate in your key vault.

At this point you should be asking: How can the VM access the key vault? Does it not require an access policy? The answer is: Yes, it does. The easiest way to do this is to enable `Managed Identity` for the Virtual Machine or Virtual Machine Scaleset. In terraform this can be achieved by specifying:

```yaml
  identity {
    type = "SystemAssigned"
  }
```

Once your VM or scale set has a system-assigned identity, you can go ahead and create the key vault policy. In terraform:

```yaml
resource "azurerm_key_vault_access_policy" "kv_access_policy" {
  provider                = azurerm.mysubscription
  key_vault_id            = data.azurerm_key_vault.kv_acmebot.id
  tenant_id               = data.azurerm_key_vault.kv_acmebot.tenant_id
  object_id               = var.vm_identity_principal_id
  certificate_permissions = ["Get","List"]
  key_permissions         = ["Get","List"]
  secret_permissions      = ["Get","List"]
}
```

The `object_id` here reffers to the system-assigned identity id of your VM or scale set. The super-important thing to remeber here, is that just creating an access policy with "Get" and "List" `certificate_permissions` will not be sufficient, The policy also requires "Get" and "List" `secret_permissions`. This is not untuitive. One would expect that storing and retrieving a TLS certificate from a key vault, the policy only requires `certificate_permissions`. However, HTTPS TLS certificate at the receiving end requires a private key to decrypt the traffic. The certificate is usually stored in PFX format and private key portion of it is stored separately as a `secret` (not a `certificate`) in a key vault. This fact is not obvious at all when interacting with the Key Vault using UI interface and will only become apparent after extensive trial/error/debugging.

## Step 3 Testing things out

The key milestones to check for during the set up process are:
- Is the Azure function able to create DNS records and place certificates in a key vault?
- Is the VM or scale set key vault extension throwing any errors? (check logs on C drive)
- Does the certificate appear in the certificate store on the virtual machine and does it contain a private key?
- Is all of this still working if you tear down and spin up a new VM/scale set? (remember, managed identity will change, new key vault access policy required)
- Is all of this still working after 60 days when the current certificate expires and renewal is required?

## Step 4 Keeping bindings up to date with new certificates

Any HTTPS bindings created in IIS need to be associated with a specific PFX certificate from the certificate store. All Let's Encrypt certificates are only valid for 90 days in total and renewal process should kick in after 60 days. The VM extension is polling the key vault in regular intervals, so when a new certificate is issued and placed in a key vault, the extension will automatically pick it up and place it in a certificate store on the VM. It will not replace the existing certificate, however. Both certificates will co-exist in the cert store. They will have identical names and subjects, but different expiry dates. So, the final step to full automation is to update any IIS bindings from pointing at the old certificate to point at the new certificate. I've written a PowerShell script to do this and set it to run as a scheduled task on the VM:

```powershell
$certSubject = "CN=*.yourdomain.com"

$certificate = Get-ChildItem Cert:\LocalMachine\My | Where-Object { $_.Subject -eq $certSubject } | Sort-Object -Property NotAfter -Descending | Select-Object -First 1

if (-Not ($certificate)) {
    Write-Host "Unable to find certificate with subject $certSubject"
    exit 1
}

Write-Host ([string]::Format("Found certificate in cert store with subject {0}, thumbprint {1}, expiring on {2}", $certificate.Subject, $certificate.Thumbprint, $certificate.NotAfter))

$bindings = Get-WebBinding | Where-Object { $_.protocol -eq "https" }

$bindings | ForEach-Object {
    $bindingInfo = $_.bindingInformation.Split(":")
    $hostHeader = $bindingInfo[2]

    $httpsBindingCert = Get-Item -Path "IIS:\SslBindings\!443!$hostHeader" -ErrorAction SilentlyContinue

    if ($httpsBindingCert) {
        Write-Host ([string]::Format("Existing binding uses certificate with thumbprint {0}", $httpsBindingCert.Thumbprint))

        if ($httpsBindingCert.Thumbprint -eq $certificate.Thumbprint) {
            Write-Host "Identical certificates, taking no action."
        }
        else {
            Write-Host "Updating certificate..."
            Remove-Item -Path "IIS:\SslBindings\!443!$hostHeader"
            New-Item -Path "IIS:\SslBindings\!443!$hostHeader" -Value $certificate -SSLFlags 1
            Write-Host "Certificate updated."
        }
    }
}
```

The powershell script will look for certificates with the specified subject, order them by expiry date and pick the latest one. It will then iterate through all HTTPS bindings in IIS and check if the thumbprint matches the latest certificate. If it does, no action is needed. If it doesn't, the HTTPS binding is updated to point at the new certificate.

## Summary

It is obvious that setting up a fully automated wildcard certificate issuing via Let's Encrypt in a Windows virtual machine world is riddled with pitfalls and it takes a while to fine-tune the whole process, so that it executes reliably every 60 days without needing any manual intervention. Having these mundane tasks fully automated is oddly satisfying though. Just don't automate yourself out of a job.