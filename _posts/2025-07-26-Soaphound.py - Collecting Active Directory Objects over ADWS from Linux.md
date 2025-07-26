# Soaphound.py - Another way of collecting AD objects from Linux - ADWS 

Authors: [@belettet1m0ree](https://x.com/belettet1m0ree), [@Sant0rryu](https://x.com/Sant0rryu)

## Introduction

Since [Soaphound](https://falconforce.nl/soaphound-tool-to-collect-active-directory-data-via-adws/) was introduced, we discovered another way to perform LDAP-like connections and queries to a Domain Controller that is both effective and less monitored. Specifically, the ADWS (Active Directory Web Services) service allows authenticated users within a domain to perform various actions on directory objects.

However, it seems - at least from our perspective - that the ADWS service is either overlooked or, at best, significantly underutilized during penetration tests and red team operations, especially when assessors aim to avoid deploying a Windows virtual machine or dropping executables on domain-joined machines during engagements.

In this blog post, we will present the ADWS service, explain how it enables the retrieval and modification of object information, and share our perspective by introducing a tool that collects Active Directory objects in a way that is fully compatible with BloodHound ingestion.


## ADWS - quèsaco? - Active Directory Web Service 

Active Directory Web Services (ADWS) is a SOAP/XML interface exposed by Microsoft to facilitate interaction with instances of Active Directory Domain Services (AD DS), Active Directory Lightweight Directory Services (AD LDS), or locally mounted AD databases via dsamain.exe. Built on the Windows Communication Foundation (WCF), ADWS exposes service endpoints compatible with SOAP clients, enabling programmatic access to directory objects and operations without requiring direct LDAP protocol implementation.

ADWS is installed by default on all Windows Server systems starting from version 2008 R2, where Active Directory role is present. It is automatically enabled when a domain controller is promoted (via **DCPromo**) or an AD LDS instance is deployed. The service listens on TCP port **9389**, and requires no additional components as long as LDAP is enabled on the system.

Authentication is supported through standard mechanisms within the Microsoft ecosystem, including Kerberos, NTLM, and Basic authentication. The service’s behavior is defined in the configuration file Microsoft.ActiveDirectory.WebServices.exe.config, located in %WINDIR%\ADWS.

Its implementation relies on Windows Communication Foundation (WCF), in alignment with the principles of Service-Oriented Architecture (SOA), developed in response to the growing need for interoperability since the 1990s. WCF enables asynchronous message-based communication using SOAP over various bindings (HTTP, TCP, NamedPipes), in compliance with WSDL and UDDL schemas.

ADWS is used by several administrative tools such as the Active Directory Administrative Center (ADAC), the Active Directory PowerShell modules, and certain internal functions of dsac.exe. When Domain controller receives an ADWS request for LDAP query, it performs internaly LDAP requests to retrieve the results. You can have a look to [WS-Enumeration](https://www.w3.org/submissions/2006/SUBM-WS-Enumeration-20060315/) for more information. 

Here is a simplified illustration of LDAP requests through ADWS below:



![LDAP requests through ADWS service]({{ site.baseurl }}/assets/img/ADWS_LDAP.png)


### Protocols overview and how it works

ADWS does not expose a traditional HTTP web service, but rather a SOAP-based interface using Net.TCP bindings. Clients communicate with the ADWS server over a dedicated NetTcpBinding session, which initiates a persistent TCP connection (default port 9389). This connection leverages WCF's binary encoding, optimizing message exchange. Access to directory data is implemented through [WS-Transfer WXFR](https://www.w3.org/submissions/2006/SUBM-WS-Transfer-20060927/) protocol as defined in [MS-ADTS](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-adts/d2435927-0999-4c62-8c6d-13ba31a52e1a), allowing programmatic interaction with AD objects.

Different services are exposed through distinct URIs, typically formatted as:


```bash
net.tcp://<hostname>:9389/ActiveDirectoryWebServices/<ServiceName>
```

Key endpoints we are interested in include:

**Enumeration** : allows to query and read directory objects.
**Resource**: allows to write or modify directory objects.

An interesting point to notice is that before all ADWS communications take place, the .NET NegotiateStream Protocol (MS-NNS) insures mutual authentication and provides encryption. The .NET NegotiateStream protocol provides a lightweight and certificate-free way to establish secure, authenticated communication over TCP. It uses SPNEGO to negotiate the appropriate security mechanis, typically Kerberos or NTLM, without requiring TLS. The protocol works in two phases:

1. **Security Context Negotiation** – The client initiates a handshake with the server using GSS-API tokens to agree on authentication, integrity, and confidentiality mechanisms.
2. **Protected Data Transfer** – Once negotiated, both parties can exchange encrypted and/or signed data over the TCP stream.

If an error occurs during negotiation or transfer, the stream is invalidated, and a new context must be established. This makes .NET NegotiateStream an efficient alternative to TLS for secure inter-process or client-server communication within Windows environments.

Once the encrypted communication is established, different types of endpoints are requested by client to have a result. In add, authentication mechanism can differ following then endpoint client is requesting to; we can see it in Microsoft [MS-ADDM](https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-addm/59205cf6-aa8e-4f7e-be57-8b63640bf9a4) paper. 

Operations ADWS allows include:

Object view is handled through WS-Enumeration operations:

> **Enumerate**: Initializes an enumeration context based on a defined search query or filter expression.
>  **Pull**: Retrieves result objects within the context of a previously established enumeration session.
>  **Renew**: Extends the expiration time of an active enumeration context to maintain its validity.
>  **GetStatus**: Returns the current expiration metadata associated with a specific enumeration context.
>  **Release**: Gracefully disposes of a given enumeration context, releasing server-side resources.

 Object manipulation is handled through WS-Transfer operations:

>  **Delete**: Removes an existing directory object.
>  **Get**: Fetches one or more attributes from a directory object.
>  **Put**: Alters one or more attributes on a target object. The `Put` operation encompasses the following actions:

>    ​	**Add**: Appends a specified value to an attribute, or creates the attribute if it does not already exist.
>    ​	**Replace**: Substitutes all existing values of a given attribute with a new set. If the attribute does not exist, it is created. If no values are provided,  the attribute is cleared.
>    ​	**Delete**: Removes specified values from an attribute, or deletes the entire attribute if no value is given. Fails if the attribute is absent.
>  **Create**: Instantiates a new directory object with the specified properties.

In addition to perform LDAP query locally on the Domain Controller and send it back to client in SOAP XML encoded trafic, collecting Active Directory Object collect through ADWS is has benefits as this activity does not register under the LDAPSearch action type in DeviceEvents, it significantly reduces its visibility in common telemetry sources, making detection more challenging.


## Existing tools

Interesting tools exist and allow to collect domain objects through ADWS service: [PingCastle](https://www.pingcastle.com/) (C#), [Soaphound](https://github.com/FalconForceTeam/SOAPHound) (C#), [ADWSProxy](https://github.com/RabobankRedTeam/ADWSProxy) (C#), [SharpADWS](https://github.com/wh0amitz/SharpADWS) (C#) and [Soapy](https://github.com/xforcered/SoaPy) (Python).

The first tool to introduce enumeration via ADWS in an offensive context was Soaphound. However, as it is mentioned in their blog post, the tool essentially performs only two major requests during its execution. 

The first is a broad enumeration of all domain objects, requesting only three LDAP attributes, with the goal of building a cache file used later to resolve object information when needed. The second request targets the collection of all domain objects again, this time requesting between 31 and 32 LDAP attributes, depending on whether the LAPS-related option is enabled. 

While this approach is efficient for large-scale enumeration, we believe the tool lacks the ability to perform fine-grained, targeted queries on individual objects even in scenarios where such precision would be necessary or more effective.

Another limitation is that enumeration through ADWS is confined to LDAP-accessible data. As a result, we cannot retrieve active user sessions on domain machines, since this information is typically obtained via RPC or WinRM rather than LDAP. Consequently, certain privilege escalation paths especially those relying on session-based attack vectors may be omitted in this context. However, this limitation may reflect a deliberate design choice to remain stealthy, as avoiding RPC or WinRM queries reduces the likelihood of detection in monitored environments especially during Red Team engagement. 

Finally, Soaphound is implemented in C#, which limits its usage to Windows environments. We considered it could be valuable to have similar capabilities available from a Linux-based system, particularly for scenarios where deploying or operating from Windows is not feasible or desirable.

## What we bring to the table

The most significant challenge was to establish a reliable method for replicating the functionality of ActiveDirectoryWebService.cs, originally designed for Microsoft platforms. Our approach involved leveraging [net-tcp-proxy](https://github.com/ernw/net.tcp-proxy) by [ERNW](https://insinuator.net/2016/08/pentesting-webservices-with-net-tcp-binding/). It is a python based library crafted to interface with .NET web services using Net.TCP binding. Notably, the tool supports protocols such as MC-NMF, MC-NMFTB, and MS-NNS, and offers a proxy mechanism for observing communications with services that require Negotiate-based encryption. However, when attempting to retrieve large volumes of data we encountered several issues likely related to the way [NBFSE](https://learn.microsoft.com/en-us/openspecs/windows_protocols/mc-nbfse) is implemented. 

Recently, X-Force released [Soapy](https://github.com/xforcered/SoaPy) a tool that provides a reliable implementation of [NBFX](https://learn.microsoft.com/en-us/openspecs/windows_protocols/mc-nbfx/9026aae7-726e-428f-b5e3-f72fda74d378), specifically designed to handle and process large volumes of ADWS requests. 

From there we have all weapons to create a Bloodhound python ingestor through ADWS service!

### Soaphound.py

While reading [Bloodhound.py](https://github.com/dirkjanm/BloodHound.py), a Linux alternative to [Sharphound](https://github.com/SpecterOps/SharpHound), we observed that object collection is performed on a case-by-case basis. Specific conditions are evaluated to determine the most relevant information for each collected object, and tailored actions are taken accordingly.

We sought to follow a similar logic while implementing Soaphound.py, aiming to collect the most valuable information during object enumeration through ADWS. In addition, as users' session on machines are not collected throught LDAP, we reused Bloodhound.py way to perform this collect. 

The tool is currently being improved to cover all specific data collection scenarios. At the time of writing, it is capable of collecting Active Directory objects via the ADWS service and retrieving remote session data similar to what BloodHound.py achieves. Alternatively, it can operate in a mode restricted to collecting only AD objects through ADWS (using option -c ADWSOnly). 

While writing this blog post, a new article from SoaPy was released, adding new features to perform more LDAP collections. Output of this new version of SoaPy can be used as an input to BOFHound. 

There is the Github project of Soaphound.py : 
# Acknowledgements and references

- [Falcon Force Team](https://falconforce.nl/soaphound-tool-to-collect-active-directory-data-via-adws/) for the initial inspiration.
- [Bloodhound.py](https://github.com/dirkjanm/BloodHound.py), for this amazing implementation of Bloodhound ingestor.
- [Microsoft](https://learn.microsoft.com/en-us/openspecs/windows_psrotocols/ms-addm/59205cf6-aa8e-4f7e-be57-8b63640bf9a4) for the official protocol documentation.
- [ERNW](https://insinuator.net/2016/08/pentesting-webservices-with-net-tcp-binding/) for the initial boost.
- [X-force Red](https://www.ibm.com/think/x-force/stealthy-enumeration-of-active-directory-environments-through-adws) for their brilliant implementation of NBFX and research insights.
- [Rabobank red team](https://rabobank.jobs/en/techblog/adws-an-unconventional-path-into-active-directory-luc-kolen/) for sharing valuable resources and operational insights.
- [Lionel SIDIBE](https://x.com/ilionels) you know what you did ;)

