## Abusing WSUS with MITM to perform ADCS ESC8 attack

---
layout: post
title:  WSUS – Abusing WSUS server misconfiguration to perform ADCS ESC8 attack to own client machine. 
category : WSUS -MITM - ADCS

tags :  Abusing WSUS with MITM to perform ADCS ESC8 attack
---

In this article we will try to demonstrate another way (**not new**) to escalate privileges on a machine by abusing WSUS server misconfiguration.

Since 2020 when Gosecure ethical hackers released misconfigured WSUS deploy exploit  [tool(https://www.gosecure.net/blog/2020/09/03/wsus-attacks-part-1-introducing-pywsus/)] pywsus, WSUS configurations are under increasing scrutiny during penetration tests in order to compromise the client machine.
Then came Gosecure's [paper](https://www.gosecure.net/blog/2021/11/22/gosecure-investigates-abusing-windows-server-update-services-wsus-to-enable-ntlm-relaying-attacks/) about investigatng abusing Windows Server Update Services (WSUS) to enable NTLM Relaying Attacks. While reading that great article, I noticed that it was left to the reader to provide a proof-of-concept of how to exploit a vulnerable WSUS server configuration in an Active Directory Certificate Service (ADCS) context to obtain NT AUTHORITY/System privileges on a domain joined machine. Then there is the goal of this blog post! 

Before diving into the subject, let's briefly talk about WSUS misconfiguration  and the ADCS vulnerability that we want to combine combine with it.


# WSUS 101


## WSUS overview

Windows Server Update Services (WSUS)  can be installed on Windows Server 2012 and later versions and enables information technology administrators to deploy the latest Microsoft product updates. WSUS can be used to fully manage the distribution of updates that are released through Microsoft Update to computers on the network.(Source [MS](https://learn.microsoft.com/en-us/windows-server/administration/windows-server-update-services/get-started/windows-server-update-services-wsus/)). WSUS server downloads and  keeps updates locally in order to provide it to domaine computers. The, a Group Policy  Object is pushed and applied to a group of domain computers that use WSUS server for their updates.
On each of these computers, the **W**indows **U**pdate **A**uto **U**pdate **Cl**lien**t** binary - wuauclt.exe looks frequently for updates by contacting the WSUS server. We can force updates to be searched and installed too by  this binary. 
wuauclt.exe talks to  WSUS server using HTTP(S) /SOAP XML web service. Which means all update procedure is done  using web service. 

## WSUS exploitation

When deploying WSUS on a domain, using SSL is recommanded not mandatory wich leads to many unsecure configurations of WSUS that we met during penetrations tests.



Due to a plugin called `jekyll-titles-from-headings` which is supported by GitHub Pages by default. The above header (in the markdown file) will be automatically used as the pages title.

If the file does not start with a header, then the post title will be derived from the filename.

This is a sample blog post. You can talk about all sorts of fun things here.

---

### This is a header

#### Some T-SQL Code

```tsql
SELECT This, [Is], A, Code, Block -- Using SSMS style syntax highlighting
    , REVERSE('abc')
FROM dbo.SomeTable s
    CROSS JOIN dbo.OtherTable o;
```

#### Some PowerShell Code

```powershell
Write-Host "This is a powershell Code block";

# There are many other languages you can use, but the style has to be loaded first

ForEach ($thing in $things) {
    Write-Output "It highlights it using the GitHub style"
}
```
