---
title: Internal domains and Certificates using Technitium and Let's encrypt
description: ""
date: 2023-12-06T14:17:50.549Z
preview: "/img/banners/Technitium_certs_banner.png"
image: /img/banners/Technitium_certs_banner.png
tags: [Technitium, DNS, Certificates, Letsencrypt]
categories: [Infrastrucutre, DNS]
---

## Internal domains and Certificates using Technitium and Let's encrypt

One of my pet peeves is to have internal domains for my services.

Some of the domains are on a TLD, publicly accessible as I have friends and family who consume the services. Others however are internal and are mostly used by me or the machines - it is much nicer to have an internal domain such as db-sg.internal.example.com rather than having an IP address which (and it will change).

This also simplifies my own management when I have applications calling the database. When I keep moving my instances between machines 6-10 times a year, it becomes daunting to update the IP of the database, or whatever other service you are using across 10 different machines.
<!-- FM:Snippet:Start data:{"id":"Affiliate warning","fields":[]} -->
> Some links may be affiliate links that keep this site running.
{: .prompt-info}
<!-- FM:Snippet:End -->

For this post we will be referring to [Technitium][Technitium], which is an easy to use DNS management solution and is actively developed. To start, head over to their blog on how to [set up a DNS server for selfhosting][Technitium setup]. 
<!-- FM:Snippet:Start data:{"id":"VPS Links","fields":[]} -->
<div class="alert alert-info" style="background-color: lavender; border: none; color: black;">
  <!--<i class="fas fa-info-circle"></i><br>--> I host majority of my cloud instances on <a href="https://cloud.hosthatch.com/a/3785" target=_blank>HostHatch VPS (Virtual Private Server) Instance</a> (In Asia) for a steal. Some of the other hosts I use are <a href="https://my.racknerd.com/aff.php?aff=9825" target=_blank>RackNerd</a> (US) and <a href="https://clients.webhorizon.net/?affid=27" target=_blank>WebHorizon</a> (Asia+Europe) VPS, and decided that it is time to move away from Linode - which is a Great service, but I am looking to reduce the billing on instances. For comparison, I save more than 50% on <a href="https://cloud.hosthatch.com/a/3785" target=_blank>HostHatch</a> compared to <a href="https://www.linode.com/lp/refer/?r=1ba3d23e62b7803c1d317de14a21a9f96a9a1b97" target=_blank>Linode</a> ($3.33 compared to $8) - Don't get me wrong, if this was an extremely (like REALLY) critical application, I would keep it on <a href="https://www.linode.com/lp/refer/?r=1ba3d23e62b7803c1d317de14a21a9f96a9a1b97" target=_blank>Linode</a>.
</div>
<!-- FM:Snippet:End -->
> The demo below is done over tailscale, however you can use any internal networking for this. I am aware that tailscale has a HTTPS certificate solution, however others do not and not everything in my infrastructure is on tailscale.
{: .prompt-info}

### Method 1 - Internal only DNS Server

The most secure method would be to have a self hosted, not publicly accessible DNS server - meaning that your ports 53/443/5353 and whatever you choose for DNS are **not** exposed to the internet.
> Note that cybertesting IS under a TLD (Top Level Domain), I am just masking it here to simulate internal resolution, those screenshots will later show the TLD so don't be surprised where that came from.
{: .prompt-info}

#### Add a primary zone

![Adding a primary zone on technitium](/img/posts/technitium_certs/technitium_-_add_primary_zone.png)
_Adding a primary zone on technitium_

![Adding DNS records on Technitium](/img/posts/technitium_certs/technitium_-_add_records.png)
_Adding DNS records on Technitium_

> This method falls short in generation of certificates trusted by Lets Encrypt as neither your webserver nor your DNS can be accessed by Lets Encrypt servers, unless you do some workarounds.
{: .prompt-warning }

### Method 2 - Unresolvable addresses on the internet

The second method, but not the secure one would be to publish records with your internal addresses (10.0.0.0/8,100.64.0.0/10,172.16.0.0/12,192.168.0.0/16) on the records on an internet facing DNS server - meaning you open your port 53/443 to the world for queries.

> This will allow others to enumerate your records and find the IP addresses that correspond to your internal services. You can decide for yourself how critical that is to you.
{: .prompt-warning }

If we go and take a look at a DNS query, we can see that it returns the IP address for the host:
![Querying A DNS record](/img/posts/technitium_certs/technitium_-_dual_A_records.png)
_Querying A DNS record_

### Method 3 - Using the Drop requests app

> If you have more than one DNS server, you will need to install the apps and apply the same configuration across all your servers.
{: .prompt-info }

Drop requests is an app you can install with Technitium, and it does exactly what the name suggests - it blocks requests from subnets, to record types and zones.
![Installing Technitium drop reuqests app](/img/posts/technitium_certs/technitium_-_drop_requests.png)
_Installing Technitium drop reuqests app_

Configuration file can be seen below:

```json
{
  "enableBlocking": true,
  "dropMalformedRequests": false,
  "allowedNetworks": [
    "127.0.0.1",
    "::1",
    "10.99.0.0/16",
    "100.64.0.0/10",
    "fd7a:115c:a1e0:b1a::/64"
  ],
  "blockedNetworks": [
  ],
  "blockedQuestions": [
    {
      "name": "int.cybertesting.net",
      "blockZone": true
    }
  ]
}
```

The configuration file, as you can see from above allows you to whitelist networks that would be allowed to bypass the "dropped requests" configuration. The default gives you quite a good explanation what you can do if you can read JSON.
The `blockedNetworks` for us stays empty as we don't need to specifically block anything, everything besides the `allowedNetworks` will be blocked. Please change the values to fit your own needs.
`blockedQuestions` refers to the queries that are going to be blocked, each query or type should be it's own object, if you are unfamiliar with JSON, I recommend taking a look at [W3 JSON Introduction][W3 JSON].
In the above example, we are blocking the entire zone int.cybertesting.net from anyone not in the `allowedNetworks`.

![Example of Technitium dropping requests on records](/img/posts/technitium_certs/technitium_-_terminal_drop_requests.png)
_Example of Technitium dropping requests on records_

I have been experimenting with the app and it seems that wildcard matching is not supported, nor blocking the responses as this app's code only checks for requests (It is in the app name...) If someone have any other insights, feel free to email me and I'll update the article.

### Method 4 - Split Horizon

You can have split horizon together with Drop Requests, this is helpful when you have some services with internal addresses and public addresses - Split Horizon allows you to change your DNS record answer based on where the request is coming from.
If a request is coming from our internal network, it'll (Namer Server) will reply with an internal IP. If a request comes from outside of our network, it'll reply with a public IP.
There are several use cases for split horizon:

1. Prevent internal network structure details from being exposed to the outside world. Internal DNS servers can contain records that are not available on the public-facing DNS servers.
2. For services hosted within a private network, an internal IP address would be returned for internal clients, facilitating direct local network access.
3. Different DNS information can be provided to clients based on their geographical location or other criteria to optimize network traffic and server load.
4. When deploying new services or testing updates, often you'd have an environment that replicates the production setup but is only accessible internally
5. If you have a VPN (such as wireguard), you can give responses that provide routes to internal resources.

If you do not have any public facing services, you can skip to [Additional considerations for internal networks](#additional-considerations).

![Installing Split Horizon App on Technitium](/img/posts/technitium_certs/technitium_-_split_horizon_app.png)
_Installing Split Horizon App on Technitium_

Once you installed it, configure the config file if you want automatic resolution, I avoided this and it is not enabled on my own setup.

```json
{
    "enableAddressTranslation": true,
    "networkGroupMap": {
        "100.64.0.0/10": "tailscale",
        "10.99.0.0/16": "wireguard"
    },
    "groups": [
        {
            "name": "tailscale",
            "enabled": true,
            "translateReverseLookups": true,
            "externalToInternalTranslation": {
               "103.xxx.xxx.xxx":"100.64.0.6",
               "134.xxx.xxx.xxx":"100.64.0.7",
               "150.xxx.xxx.xxx":"100.64.0.8"
            }
        } 
    ]
}
```

In the configuration above, we define the networks and their mappings, this is a very static mapping though. To make this a bit more robust we are going to use the APP records available in Technitium, which is something I prefer.
![Configuring Split Horizon on Technitium](/img/posts/technitium_certs/technitium_-_app_record_split_horizon.png)
_Configuring Split Horizon on Technitium_

The screenshot below will show you that I am able to resolve the host, that is because I disabled the Drop Requests app for **demo sake**, if you enabled the app, your resolution should not be successful on the public facing interface.
In the screenshot below, ns1.cybertesting.net is the public nameserver for the domain, facing the internet. When we do a NS lookup on the public facing IP we get the record of the public one, when we do that to the internal IP we get the internal address.

In the screenshot below, ns1.cybertesting.net is the public nameserver for the domain, facing the internet. When we do a NS lookup on the public facing IP we get the record of the public one, when we do that to the internal IP we get the internal address.
![What a dns query looks like with split horizon](/img/posts/technitium_certs/technitium_-_nslookup_split_horizon.png)
_What a dns query looks like with split horizon_

Make sure that you have configured your zone transfer settings correctly (default should be good enough) and you do not allow zone transfers without a TSIG key, if you haven't then someone can do a zone transfer and decode the split horizon record - we will also need this for later on in the certificates.
![Running a AXFR request on an unprotected zone](/img/posts/technitium_certs/technitium_-_dig_zone_transfer.png)
_Running a AXFR request on an unprotected zone_

The hex string with the TYPE65282:

```text
010D53706C697420486F72697A6F6E1A53706C6974486F72697A6F6E
2E53696D706C65416464726573734F7B0A2020227075626C6963223A
205B0A2020202022382E382E382E38220A2020205D2C0A2020223130
302E36342E302E302F3130223A205B0A20202020223130302E36342E
302E36220A20205D0A7D
```

Would decode to:

```text
.Split Horizon.SplitHorizon.SimpleAddress{
  "public": [
    "8.8.8.8" 
  ],
  "100.64.0.0/10": [
    "100.64.0.6" 
  ]
}
```

That is how it should work as you do need to facilitate the transfer of zones between nameservers, so make sure to use TSIG keys and the zone transfer control. 
![Zone configuration on Technitium](/img/posts/technitium_certs/technitium_-_zone_options.png)
_Zone configuration on Technitium_

### Additional Considerations

You don't actually have to set a "public" network for split horizon, what you could do is leave it empty and anyone looking it up will send back a blank answer.
Configuration would look like this for the record configuration:

```json
{
  "public": [
    ""
   ],
  "100.64.0.0/10": [
    "100.124.17.4"
  ]
}
```

<br>
![A blank record when querying split horizon](/img/posts/technitium_certs/technitium_-_split_horizon_blank_A_record.png)
_A blank record when querying split horizon]_

Alright, so now we have resolving done on records we can continue to certificates!

### Let's Encrypt certificates using RFC2136 and Certbot

There are basically two ways to do these, we can either configure an API token on Technitium or use RFC2136 for the DNS-01 challenge, Technitium published a [blogpost here](https://blog.technitium.com/2023/03/how-to-auto-renew-ssl-certificates-with.html). 
I will not be covering the HTTP-01 challenge as your webserver will not be resolvable if you use internal addressing, as such we can't verify it unless we go through various workarounds on redirections. Hence I'll stick to DNS-01 challenge for now using RFC2136.

I will not be covering the HTTP-01 challenge as your webserver will not be resolvable if you use internal addressing, as such we can't verify it unless we go through various workarounds on redirections. Hence I'll stick to DNS-01 challenge for now using RFC2136.

#### RFC2136 Use Cases

- **Dynamic IP Address Assignment:** In networks where IP addresses are dynamically assigned (like with DHCP), RFC 2136 enables the automatic update of DNS records when IP addresses change.
- **Service Registration:** Services that need to register themselves in DNS (like certain types of clustered services or load balancers) can use dynamic updates for real-time DNS record management.

#### RFC 2136 Traffic Flow in DNS Update

1. **Update Request:** The client (which could be a DHCP server, a service, or an administrator's tool) sends an update request to the DNS server. This request contains all the necessary information in the format specified by RFC 2136.
2. **Authentication and Authorization:** The DNS server authenticates the request (if security mechanisms are in place) and checks if the client is authorized to make the requested changes.
3. **Processing Prerequisites:** The DNS server checks the prerequisites section of the update message to ensure all conditions are met before proceeding with the update.
4. **Applying the Update:** If all prerequisites are met, the server applies the changes listed in the update section. This might involve adding, deleting, or modifying DNS records in the zone.
5. **Response:** The DNS server sends a response back to the client indicating the success or failure of the update request.
6. **Propagation:** Depending on the DNS infrastructure, the update may need to be propagated to other DNS servers, like secondary servers in a zone.

#### Hands-on

If you prefer to follow the Technitium guide, it is linked at the top of the section.
For this demo I will be using Technitium itself to update the certificate of the web interface which is blocked from the outside - do note, that for this to work without many workarounds you will need to have your DNS server facing the internet.
Install certbot and rfc2136 plugin:

```shell
sudo apt update
sudo apt install certbot python3-pip -y
sudo python3 -m pip install certbot-dns-rfc2136
```
{: .nolineno }

You'll need to create a TSIG key that will allow the plugin to create the appropriate records.
![Adding a TSIG Key on Technitium](/img/posts/technitium_certs/technitium_-_TSIG_key.png)
_Adding a TSIG Key on Technitium_

Load the TSIG key in the configuration of the zone:
![Configuring RFC2136 Dynamic updates on Technitium](/img/posts/technitium_certs/technitium_-_RFC2136_Configuration.png)
_Configuring RFC2136 Dynamic updates on Technitium_

> I use *.int.cybertesting.net as during my testing as I issue per host certificates and seems like _acme-challenge.*.cybertesting.net does not work. 
Seems as Technitium does not allow wildcard in the middle of the domain.
{: .prompt-info}

Once these two are done, create and update your certbot-rfc2136.ini file.

```shell
sudo nano /root/certbot-rfc2136.ini
```
{: .nolineno }

Paste the following in:

```text
# Target DNS server (IPv4 or IPv6 address, not a hostname)
dns_rfc2136_server = 192.0.2.1
# Target DNS port
dns_rfc2136_port = 53
# TSIG key name
dns_rfc2136_name = KEY-NAME_HERE.
# TSIG key secret
dns_rfc2136_secret = TSIG-KEY-HERE
# TSIG key algorithm
dns_rfc2136_algorithm = HMAC-SHA512
# TSIG sign SOA query (optional, default: false)
dns_rfc2136_sign_query = false
```
{: file="/root/certbot-rfc2136.ini" }





{% include links.md %}
[Technitium setup]: https://blog.technitium.com/2022/06/how-to-self-host-your-own-domain-name.html
[W3 JSON]: https://www.w3schools.com/js/js_json_intro.asp
