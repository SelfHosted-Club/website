---
title: Migrating Uptime Kuma on Docker
description: "Migrate your UptimeKuma monitoring stack to a new hosting provider. This guide provides step-by-step instructions to move your container and preserve all data. "
date: 2023-11-22T14:00:27.227Z
preview: "https://images.unsplash.com/photo-1526072339703-f3253bc87fdc?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=M3wxMTc3M3wwfDF8c2VhcmNofDV8fG1pZ3JhdGV8ZW58MHx8fHwxNjk5NTQxNDI1fDA&ixlib=rb-4.0.3&q=80&w=960"
image: https://images.unsplash.com/photo-1526072339703-f3253bc87fdc?crop=entropy&cs=tinysrgb&fit=max&fm=jpg&ixid=M3wxMTc3M3wwfDF8c2VhcmNofDV8fG1pZ3JhdGV8ZW58MHx8fHwxNjk5NTQxNDI1fDA&ixlib=rb-4.0.3&q=80&w=960
tags: [UptimeKuma, Infrastructure]
categories: [Docker, Monitoring]
---

## Migrating Uptime Kuma on Docker

Sometimes there is a need to migrate servers, such is the case for my instance of Uptime Kuma and this post.
As I just grabbed another instance at [HostHatch][HostHatch]{:target="_blank"}, I have decided to move away my non critical workloads away from Linode.

<!-- FM:Snippet:Start data:{"id":"Affiliate warning","fields":[]} -->
> Some links may be affiliate links that keep this site running.
{: .prompt-info}
<!-- FM:Snippet:End -->

Anyway, as that is time to migrate, I'd like to take my historic data and configurations and move those over so that I don't need to adjust the scripts I have running.
We have discussed all the configurations in UptimeKuma [Part 1][Part 1] & [Part 2][Part 2], I did cover that I use a cloudflare tunnel, and so the only change you need to do there is change the pointer of your CNAME/A to the new IP once this migration is done.

<!-- FM:Snippet:Start data:{"id":"VPS Links","fields":[]} -->
<div class="alert alert-info" style="background-color: lavender; border: none; color: black;">
  <!--<i class="fas fa-info-circle"></i><br>--> I host majority of my cloud instances on <a href="https://cloud.hosthatch.com/a/3785" target=_blank>HostHatch VPS (Virtual Private Server) Instance</a> (In Asia) for a steal. Some of the other hosts I use are <a href="https://my.racknerd.com/aff.php?aff=9825" target=_blank>RackNerd</a> (US) and <a href="https://clients.webhorizon.net/?affid=27" target=_blank>WebHorizon</a> (Asia+Europe) VPS, and decided that it is time to move away from Linode - which is a Great service, but I am looking to reduce the billing on instances. For comparison, I save more than 50% on <a href="https://cloud.hosthatch.com/a/3785" target=_blank>HostHatch</a> compared to <a href="https://www.linode.com/lp/refer/?r=1ba3d23e62b7803c1d317de14a21a9f96a9a1b97" target=_blank>Linode</a> ($3.33 compared to $8) - Don't get me wrong, if this was an extremely (like REALLY) critical application, I would keep it on <a href="https://www.linode.com/lp/refer/?r=1ba3d23e62b7803c1d317de14a21a9f96a9a1b97" target=_blank>Linode</a>.
</div>
<!-- FM:Snippet:End -->

### Let's get to it
#### Old host

If you used a custom compose file that maps the data directory to your local directory, you just have to copy it over and you are done! However, this is not the case with [Linode][Linode]{:target="_blank"}, where the data folder is sitting on a docker volume.
<!-- FM:Snippet:Start data:{"id":"Shell code block","fields":[]} -->
```shell
docker volume ls
```
{: .nolineno }
<!-- FM:Snippet:End -->

The command above will list all the docker volumes we have, we are looking specifically for the uptime-kuma volume.

!["UptimeKuma docker volume"](/img/posts/uptimekuma/UptimeKuma_-_Docker_Volume.png)
_UptimeKuma docker volume_

Once this is done, we need to export somehow the data on the volume out, I have experimented with the various commands, and found that tarring the directory inside the docker, then extracting it on the other side offered ease of use compared to using the built in export command, even though it is more commands to type at the end.

On the old host, type in:
<!-- FM:Snippet:Start data:{"id":"Shell code block","fields":[]} -->
```shell
docker exec uptime-kuma tar cvf /tmp/backup-uptimekuma.tar /app/data
```
{: .nolineno }
<!-- FM:Snippet:End -->

This will create a tar of the `/app/data` directory and save it inside the container in the `/tmp` directory.
We will then copy out the file from the container to our host by running the built-in `docker cp` command as follows:
<!-- FM:Snippet:Start data:{"id":"Shell code block","fields":[]} -->
```shell
docker cp uptime-kuma:/tmp/backup-uptimekuma.tar ~/backup-uptimekuma.tar
```
{: .nolineno }
<!-- FM:Snippet:End -->

!["UptimeKuma export volume"](/img/posts/uptimekuma/UptimeKuma_-_Export_data_folder.gif)
_UptimeKuma export volume_

Once that is done, you need to transfer this file to your new host - if that is not a critical application (which for us it isn't), we would recommend you host it on either [HostHatch][HostHatch],[WebHorizon](https://clients.webhorizon.net/?affid=27) or [RackNerd](https://my.racknerd.com/aff.php?aff=9825), for more critical ones do checkout [Linode][Linode] or your favourite enterprise provider, though there is a small premium for it.

If you have a cloudflare tunnel running I'd recommend to stop the container using `docker stop uptime-kuma`.

> Do not delete your running instance until you confirm everything on the new one is in order and working!
{: .prompt-danger }


#### New host

 I like to manage the docker infrastructure through docker compose files, and so for this I created a new one which is detailed below:
 ```yaml
 version: '3'

services:

  uptime-kuma:
    container_name: uptime-kuma
    hostname: uptime-kuma
    image: louislam/uptime-kuma:1
    volumes:
      - uptime-kuma:/app/data
    restart: always

volumes:
  uptime-kuma:
    name: uptime-kuma
```
{: file="/docker/uptimekuma/docker-compose.yaml" }

Once that is done we are going to bring up the container using the `docker compose up -d` command. Now if you had followed previous guides, we had a cloudflare tunnel up and running, you can keep that tunnel up, just make sure to bring down the old host if you haven't done so.

We will now copy over the tar into the container, and extract the file inside of it using:
<!-- FM:Snippet:Start data:{"id":"Shell code block","fields":[]} -->
```shell
docker cp ~/backup-uptimekuma.tar uptime-kuma:/tmp/backup-uptimekuma.tar
docker exec uptime-kuma tar xvf /tmp/backup-uptimekuma.tar -C /
```
{: .nolineno }

All that is left is to restart the container and we are good to go!
<!-- FM:Snippet:Start data:{"id":"Shell code block","fields":[]} -->
```shell
docker compose down && docker compose up -d
```

{: .nolineno }

!["UptimeKuma importing volume"](/img/posts/uptimekuma/UptimeKuma_-_Import_data.gif)
_UptimeKuma importing volume_

If you are using A records, just point your DNS to the new IP of your server.
> If you are using the previous scripts we published in [Uptime Kuma - Part 1][Part 1] then you just need to update your DNS records or do nothing if you use a tunnel.
If you are using IP, then you'll need to update all your scripts.
{: .prompt-tip }

You can go ahead and check the logs for any issues you might have by using the command `docker compose logs -f` .

<!-- FM:Snippet:End -->

Cover image credits go to [Jeffery Hamilton](https://unsplash.com/@pistos?utm_source=ghost&utm_medium=referral&utm_campaign=api-credit) / [Unsplash](https://unsplash.com/?utm_source=ghost&utm_medium=referral&utm_campaign=api-credit)

[Part 1]: /posts/service-and-infrastructure-monitoring-with-uptime-kuma-part-1/
[Part 2]: /posts/service-and-infrastructure-monitoring-with-uptime-kuma-part-2/

{% include links.md %}
