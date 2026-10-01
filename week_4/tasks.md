## Discovering ASNs

### General Information

Discovering the AS number of an organization allows us to scan the entire IP range allocated to that organization.

Not every organization has an AS number, but it is nice when they do.

> When discovering an ASN, make sure the result is not a false-positive (i.e. out-of-scope)

### Challenge: Discover the ASN that tue.nl belongs to

<details>
<summary>Hint</summary>
Use https://bgp.he.net/ and lookup tue.nl
</details>

<details>
<summary> Answer </summary>
AS1161 (SURV B.V.)
</details>

> Does the ASN you discovered really belong to TU/e?

### Challenge: Discover the ASN that municipality Amsterdam uses and it's IP range

<details>
<summary>Answer</summary>
ASN: AS51647 (Gemeente Amsterdam)<br><br>
IP Range: 46.17.24.0/21
</details>

> Do you think municipality Amsterdam owns everything inside this IP range?

## Discovering Apex Domains

### General Information

**Apex domain:** An apex domain is the domain name itself with no subdomain in front: `example.com` rather than `www.example.com`.

Every organization that uses Microsoft365 is a tenant with a specific ID called a tenant ID.

A tenant ID is an identifier that gets assigned to a company by Microsoft.

A tenant can have multiple domains registered under the same tenant ID. So, the tenant "TU Eindhoven" can have the tenant ID `cc7df247-60ce-4a0f-9d75-704cf60efc64` and the following domains might be registered under that tenant ID:

```
barcommissie.nl
dpivaluecenter.nl
dpivaluecentre.nl
eindhovenengine.nl
...
techniekpromotie.nl
tue.nl
```

This means that all of domains are managed by TU Eindhoven. If you manage to hack one of these domains, that means you hacked "TU Eindhoven".

### Challenge: Find 4 apex domains that are registered under Nijmegen's tenant

<details>
<summary>Hint</summary>
You can use https://micahvandeusen.com/tools/tenant-domains/

</details>

<details>
<summary>Answer</summary>

Just search "nijmegen.nl" on https://micahvandeusen.com/tools/tenant-domains/.

Results:<br><br>

ggibnijmegen.nl <br><br>
nijmegen.nl <br><br>
waalsprong.nl <br><br>
wijzijngroengezondeninbewegingnijmegen.nl <br><br>

</details>

## Discovering Sub Domains

### General Information

**Subdomain:** A subdomain is an additional part placed in front of the apex domain: `www.example.com` is a subdomain of `example.com`.

Organizations often use subdomains to separate different services, applications, or environments while keeping them under the same main domain.

For example, TU Eindhoven might use subdomains such as:

```
www.tue.nl
mail.tue.nl
portal.tue.nl
vpn.tue.nl
api.tue.nl
...
```

Each of these is a subdomain of the apex domain `tue.nl`.

Subdomains can point to completely different servers or services. For example, `www.tue.nl` could host the organization's public website, while `mail.tue.nl` could point to a mail service and `vpn.tue.nl` could point to a VPN service.

This means that discovering subdomains can provide additional information about an organization's attack surface. A forgotten, misconfigured, or vulnerable subdomain may expose a service that is not immediately visible when looking only at the organization's main website.

It is important to note that a subdomain does not necessarily represent a separate organization or tenant. Multiple subdomains can be part of the same infrastructure, and some may be operated by third-party providers.

Therefore, discovering subdomains is useful during reconnaissance because it can reveal additional services, applications, and infrastructure associated with an organization.

### Challenge: Discover ~20 subdomains under nijmegen.nl

<details>
<summary>Hint</summary>
Use crt.name
</details>

<details>
<summary> Answer </summary>
No answer, it's a free-for-all
</details>

### Challenge: Discover a subdomain that hosts the test version of a production application by municipality Nijmegen. This version has a feature that the real application does not.

<details>
<summary> Hint 1</summary>
The login page has a guest/anonymous account login in the test version
</details>

<details>
<summary>Hint 2</summary>
Look for keywords like 'test' in discovered subdomains
</details>

<details>
<summary>Answer</summary>
parkeerproducten-test.nijmegen.nl -> test

parkeerproducten.nijmegen.nl -> production/real service

</details>

### Challenge: Discover the subdomain that hosts Nijmegen's FTP service

<details>
<summary>Answer</summary>
ftp.nijmegen.nl
</details>

### Challenge: Discover the second apex domain that is associated with this FTP service

<details>
<summary>Hint 1</summary>
Look at the SSL certificate data
</details>

<details>
<summary>Hint 2</summary>
Use Shodan (creating an account makes it easier)
</details>

<details>
<summary>Answer</summary>
After discovering ftp.nijmegen.nl, look up it's IP address. <br>
You can do "dig ftp.nijmegen.nl" on Linux or use an online <br>
DNS lookup tool. <br>

After finding the IP (145.11.60.40) paste it to Shodan.

Discovered: irvn.nl

</details>

>  Can you tell if the newly found domain likely to be in scope during Nymacon?

### 

### Challenge: Find the subdomain that hosts the VPN service of the second apex domain

<details>
<summary> Answer </summary>

vpn.irvn.nl

</details>

### Challenge: Find the PoC of a vulnerability that this service was affected by.

<details>
<summary>Hint</summary>
Look up if the vpn service has any CVEs, then look <br>
up if PoCs have been released on GitHub for those CVEs.
</details>

## Staying in Scope

### General Information

Sometimes a recon can go out-of-scope. You think you find an ASN that belongs to an organization when it doesn't. Or you do an IP lookup using an domain by the organization (irvn.nl -> )

### Challenge: If you attack the IP address that ftp.irvn.nl resolves to, are you in the same scope as ftp.irvn.nl?

<details>
<summary> Hint 1 </summary>
Is irvn.nl using a cloud provider?
</details>

<details>
<summary> Hint 2 </summary>
How do cloud providers use the same IP to host multiple domains.
</details>

<details>
<summary> Answer </summary>
No, the IP address of ftp.irvn.nl is owned by Vodafone. <br>
They host different services on that IP (e.g. MySQL) than <br>
what is hosted on ftp.irvn.nl (e.g. FTP)
</details>
