---
title: Service and infrastructure monitoring with Uptime Kuma - Part 1
description: "Discover how UptimeKuma, an open-source and self-hosted uptime monitoring tool, empowers you to stay ahead of server or service failures."
date: 2023-06-06T17:02:15.293Z
author: Tivin
image: /img/banners/UptimeKuma_part1.jpeg
preview: /img/banners/UptimeKuma_part1.jpeg
tags: [UptimeKuma, Infrastructure]
categories: [Docker, Monitoring]
---

**Part 2 is [available here][Part 2]**

**Migration guide for UptimeKuma is [here](https://www.selfhosted.club/migrating-uptimekuma/)**
## Service and infrastructure monitoring with Uptime Kuma - Part 1

From time to time servers or services might fail, and as a selfhoster you need to know when that happens. [Uptime Kuma](https://github.com/louislam/uptime-kuma?ref=selfhosted.club) is one of those extremely helpful apps that will you know when such a thing happens. It is (as the name suggests) and uptime monitoring tool that can help you. Uptime monitoring can also help you understand if a platform or an app you are subscribing to are fulfilling their SLA (Service Level Agreement); and Uptime Kuma does that extremely well.


![Uptime Kuma Monitors](/img/posts/uptimekuma/UptimeKuma_Monitors.png)
_The various monitors offered by UptimeKuma_

**HTTP(S) Monitor:** This is probably the most common type of monitor you'll be using. This monitor checks the status of a web service by sending a HTTP or HTTPS request to a specified URL. You can configure it to consider a certain HTTP status code (like 200 OK) as "up," and any other code as "down." You can also set up advanced options like request method (GET, POST, etc.), custom headers, and expected response body as well as certificate monitoring.

![UptimeKuma HTTP Monitor](/img/posts/uptimekuma/UptimeKuma_HTTP_Monitor.gif)
_UptimeKuma HTTP Monitor walkthrough_

> Some links may be affiliate links that keep this site running.
{: .prompt-info}


**TCP Monitor:** This monitor is used to check the status of a TCP service, such as a database or email server. You specify the IP address or hostname and the port number, and Uptime Kuma will try to establish a TCP connection. If the connection is successful, the service is considered "up," otherwise, it's "down."

**Ping Monitor:** This monitor sends an ICMP "echo request" (also known as a ping) to an IP address or hostname. If it receives an echo reply, the host is considered "up," otherwise, it's "down." This monitor is useful for checking the status of network devices like routers and switches.

**DNS Monitor:** This monitor checks the status of a DNS server by making a DNS query to a specified domain. If it gets a valid DNS response, the server is considered "up," otherwise, it's "down."

**HTTP(s) Keyword Monitor:** This monitor is similar to the HTTP(S) monitor, but instead of checking the HTTP status code, it checks the response body for a specific keyword or phrase. If the keyword is found, the service is considered "up," otherwise, it's "down." This can be useful for checking if a web service is not just responding, but responding correctly.

**gRPC(s) - Keyword:** The gRPC keyword monitor is a proposed feature in Uptime Kuma that aims to enable monitoring of internal services that only support the gRPC protocol.

**Docker container:** This monitor allows you to monitor the status of docker containers by connecting to the docker host either through the socket (which you'll need to expose) or through the web service Docker can be configured for monitoring.

<!-- FM:Snippet:Start data:{"id":"Affiliate Links","fields":[]} -->
<div class="alert alert-info" style="background-color: lavender; border: none; color: black;">
  <!--<i class="fas fa-info-circle"></i><br>--> I host majority of my cloud instances on <a href="https://cloud.hosthatch.com/a/3785" target=_blank>HostHatch VPS (Virtual Private Server) Instance</a> (In Asia) for a steal. Some of the other hosts I use are <a href="https://my.racknerd.com/aff.php?aff=9825" target=_blank>RackNerd</a> (US) and <a href="https://clients.webhorizon.net/?affid=27" target=_blank>WebHorizon</a> (Asia+Europe) VPS, and decided that it is time to move away from Linode - which is a Great service, but I am looking to reduce the billing on instances. For comparison, I save more than 50% on <a href="https://cloud.hosthatch.com/a/3785" target=_blank>HostHatch</a> compared to <a href="https://www.linode.com/lp/refer/?r=1ba3d23e62b7803c1d317de14a21a9f96a9a1b97" target=_blank>Linode</a> ($3.33 compared to $8) - Don't get me wrong, if this was an extremely (like REALLY) critical application, I would keep it on <a href="https://www.linode.com/lp/refer/?r=1ba3d23e62b7803c1d317de14a21a9f96a9a1b97" target=_blank>Linode</a>.
</div>
<!-- FM:Snippet:End -->

![Setting up docker monitor](/img/posts/uptimekuma/UptimeKuma_dockerhost_monitor.gif)
_Setting up Docker monitor_

**Push:** This is probably one of my favourite monitors and it is a passive one, it creates a listener on Uptime Kuma and waits for a call; you can customise the messages and and add other info it. If you are familiar with [healthchecks.io][healthchecks.io], you can achieve something similar here.

What I have found through the years of working with various devices is that a device can still respond to ping as the network stack is still up and available, it is in actuality frozen on the OS level. Because of that my preference is to use push monitors when I am able to. You can see the example below.

![UptimeKuma Push Monitor](/img/posts/uptimekuma/UptimeKuma_PlexServer_monitoring.gif)
_Setting up a push monitor_

Here is the code for the shell script to push back a ping of the server as well, you will need to install jq first to work a bit easier with JSON data:

Install jq:

```shell
sudo apt install jq
```

Create a script file such as healthcheck.sh and paste the code below to it, you can run that or create a cron for that.
```shell
#!/bin/bash

# Define the URL for the curl command
url="<your url here up to the ? mark>"
#Test IP to run the PING command against
test_ip="1.1.1.1"

# Execute the ping command and capture the output
ping_output_rtt=$(ping -c 1 $test_ip | awk '/rtt/ {print $4}' | cut -d '/' -f 2)
ping_output_round_trip=$(ping -c 1 $test_ip | awk '/round-trip/ {print $4}' | cut -d '/' -f 2)

# Use whichever output is not empty
if [ -z "$ping_output_rtt" ]
then
    ping_output=$ping_output_round_trip
else
    ping_output=$ping_output_rtt
fi

# URL-encode the ping output
encoded_ping_output=$(printf "%s" "$ping_output" | jq -s -R -r @uri)

# Construct the complete curl command with the ping value
curl_command="curl -m 10 --retry 5 '${url}?status=up&msg=OK&ping=${encoded_ping_output}'"

# Execute the curl command
eval "$curl_command"
```

Another interesting thing you can do if you change the script a bit, and create a service monitoring script for various services you run and might want to monitor.

![UptimeKuma Service Monitoring](/img/posts/uptimekuma/UptimeKuma_PlexServiceMonitoring.gif)
_Setting up a service monitor_

Here is the code for the service monitoring script:
```shell
#!/bin/bash

# Define the URL for the curl command
url="<your url here up to the ? mark>"

# Define the service name
SERVICE_NAME="<The service name as it is under systemd>"

# Check the status of the service
if systemctl is-active --quiet "$SERVICE_NAME"
then
    # If the service is running, set the status to 'up' and the message to 'OK'
    status="up"
    msg="OK"
else
    # If the service is not running, set the status to 'down' and the message to 'Service is not running'
    status="down"
    msg="Service is not running"
fi

# URL-encode the message
encoded_msg=$(printf "%s" "$msg" | jq -s -R -r @uri)

# Construct the complete curl command with the status and message
curl_command="curl -m 10 --retry 5 '${url}?status=${status}&msg=${encoded_msg}'"

# Execute the curl command
eval "$curl_command"
```

The rest of the monitors are very specific and well... quite self explanatory, we will document and record those in future.

### The use of tags in Uptime Kuma

You have the possibility to assign tags to your monitored assets, while it is a great functionality, it is a bit cumbersome on how the assignment happens. Each and every asset has to be manually edited and assigned tags; Later though you are able to use those to filter your assets in the search if you have many of those - after some time, you will have quite a few assets and services being monitored and so it is highly recommended to use tags.

![UptimeKuma Tags](/img/posts/uptimekuma/UptimeKuma_Using_Tags.gif)
_Using Tags_

### Status pages

We will finish this post with status pages, which are a valuable feature in Uptime Kuma that allow users to share the status of their monitored services publicly. Essentially, they are public-facing web pages that provide real-time information about the uptime and availability of your services. Status Pages can be customised to show the services you want, making it a fantastic tool for transparency and communication with your user base.

You can customise it showing your tags if these are internal pages, assign domain url to the page, custom CSS and Google analytics; the status will automatically change according to the signals received from your monitors.

![UptimeKuma Status Pages](/img/posts/uptimekuma/UptimeKuma_Statuspages.gif)
_Status pages_

## A few notes

I do recommend to choose a highly available host for Uptime Kuma so you may get notifications in a timely matter, for some monitors you'd might want to run those over an encrypted network such as Wireguard, OpenVPN or our favorite NetBird (which runs wireguard).

## Wrapping up

We will continue with another post on a few more details we did not touch upon right now such as the specific service monitors, notification setups, reverse proxies, cloudflare tunnel and using the API of Uptime Kuma.  
  
I hope this was beneficial, and does help you get started with Uptime monitoring, do subscribe to get new articles to your email.

Click here to read [part 2][Part 2]


[healthchecks.io]: https://healthchecks.io/?ref=selfhosted.club
[HostHatch]: https://cloud.hosthatch.com/a/3785?ref=selfhosted.club
[RackNerd]: https://my.racknerd.com/aff.php?aff=9825&ref=selfhosted.club
[WebHorizon]: https://clients.webhorizon.net/?affid=27&ref=selfhosted.club
[Linode]: https://www.linode.com/lp/refer/?r=1ba3d23e62b7803c1d317de14a21a9f96a9a1b97&ref=selfhosted.club
[Part 2]: /posts/service-and-infrastructure-monitoring-with-uptime-kuma-part-2/
