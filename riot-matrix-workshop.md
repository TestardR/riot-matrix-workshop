# Running your own encrypted chat service with Matrix and Riot

**Authors and Workshop Instructors:**

- Lilly Ryan [@attacus_au](https://twitter.com/attacus_au)
- Gabor Szathmari [@CryptoAustralia](https://twitter.com/CryptoAustralia)

This workshop is distributed under a [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) license.

## What are we doing here?

The goal of this workshop is to teach you how to configure and run your own Matrix/Riot service. By the end of the workshop, you should be able to log into secure chat rooms and invite others to the same server.

Don't panic if you don't finish all of these steps today. You can take these instructions with you and continue/start again at home any time you like.

## What are Matrix and Riot?

**Disclaimer: the instructors are not getting paid by anyone or receiving any incentives to say any of this. We're telling you about it because we like it, and because we think you might find it useful.**

Matrix and Riot work together to provide a chat service which behaves in a similar way to popular services like Slack. Unlike Slack, however, this tech stack is free and open source. This means you, your team, or your company can easily host your own Matrix servers so that all information that passes through there can remain within the control of your organisation instead of a third party. **If you are frequently sharing sensitive things like passwords, memory dumps, or internal URLs as part of your bug bounty activities or penetration testing engagements - this workshop is for you!**

With Matrix and Riot, you can run chatbots, integrate Giphy, create new channels for different topics, or start one-to-one conversations. You can also configure audio and video calling. The stack features Slack and IRC integration for those who don’t want to move (there’s always at least one: https://xkcd.com/1782/).

Matrix also provides native support for encrypted chat rooms. If you set it up, chat channels on your server can be end-to-end encrypted. Anyone on your server can verify their fingerprints with each other out-of-channel to make sure that the people in the room are the ones you want there.

**Please remember** that, as with any encrypted messaging service, the room is only as secure as the people in it. Encryption isn't magic. Don't write anything you wouldn't want to see your name against publicly.

## Pre-requisites:

- A computer
- Internet connection (BYO or the conference guest wifi. If you're using the guest wifi, please be kind.)
- Basic knowledge of the Linux terminal
- Riot client installed on your laptop or phone (get it at https://riot.im/)

## Optional pre-requisites

If you want to take your work home with you, you'll also need to supply:

- Your own domain name
- VM running Debian 8 on a cloud service

# Installation Guide

## DNS settings

**Note:** if you just want to learn how Matrix and Riot work today and don't want to mess around with DNS config just yet, ask one of the instructors for temporary domain details, and skip straight to "Installing the Matrix server".

Please remember that all temporary workshop domains and VMs will not live long after the workshop; to run your own service in the real world, you'll need to set up your own infra.

### Register a domain

If you don't already have a domain you want to use for this, you can buy one through any domain registrar. If you didn’t set one up before the workshop, it’s going to take too much setup time right now to work with it in this session, so go ahead and ask for one of our temporary hostnames and come back to this later.

(If you don’t have a registrar of choice, we recommend `serversaurus.com.au` - they're awesome local folk who care about privacy. Again, we are not being paid to say that, they’re just nice people.)

### Add DNS records to your DNS panel

e.g.:

```
server01.chat.cryptoa.us. | 300    IN    | A    | 1.2.3.4
_matrix._tcp.server01.chat.cryptoa.us. |    300 IN | SRV | 10 0 443 server01.chat.cryptoa.us.
```

## Installing the Matrix server

The following guide will set up Synapse, which is Matrix's homeserver implementation.

### Disclaimer

**If you're using one of our VMs, be considerate. We're hosting you so you can learn, not so you can show off. The [BSides code of conduct](http://www.bsidesau.com.au/contact.html) applies in this workshop and on our VMs just as much as it does to the rest of the event. Be nice.**

### Preparing the machine

First: launch your Debian 8 VM, and SSH in. If you didn't bring your own, you can use one of our temp VMs. Ask instructors for details.

The Matrix/Synapse package lives in a non-standard repository. We're going to add this repository to our machine's list of sources:

```
echo 'deb http://ftp.debian.org/debian jessie-backports main' >> /etc/apt/sources.list
```

And then we're going to make sure the machine knows that the repo is there:

```
apt-get update && apt-get dist-upgrade -y
```

Next, we need to install a few packages that will help us later. Our VMs are very barebones by default. Run the following:

```
apt-get install -y apt-transport-https lsof curl python python-pip
apt-get install -y certbot -t jessie-backports
```

At this point we need to add a configuration file to ensure that Matrix knows where to read from.
The file needs to be called `/etc/apt/sources.list.d/matrix.list`

Open this up in your favourite text editor. We are an equal opportunity workshop and take no sides in the text editor wars. If you don't have an editor of choice and want some guidance, ask one of the instructors (or the person next to you) for a recommendation.

Inside `/etc/apt/sources.list.d/matrix.list`, add the following two lines:

```
deb https://matrix.org/packages/debian/ jessie main
deb-src https://matrix.org/packages/debian/ jessie main
```

## Installing Matrix

With that out of the way, it's time to actually install Matrix. Run the following:

```
curl https://matrix.org/packages/debian/repo-key.asc | apt-key add -
apt-get update
apt-get install matrix-synapse -y
```

#### Warning:

You might get a `python-cffi` package conflict error at this point which will cause the `matrix-synapse` install to fail. If this is the case, we'll install Aptitude (an alternative Debian package manager) to help us resolve the conflict semi-painlessly.

Run `apt-get install aptitude`.

Try installing the package again with Aptitude: `aptitude install python-cffi`

When Aptitude offers to resolve the conflict, accept the solution which says:

```
Install the following packages:
1) python-cffi [1.4.2-2~bpo8+1 (jessie-backports)]
```

This will be the third or fourth option Aptitude gives you. Choose "no" until you see this solution offered.

After this, `python-cffi` should install properly.

Once that's done, let's try the Matrix install again:

`apt-get install matrix-synapse`

If it's still not working, we suggest a sacrifice to the apt-get gods, or asking an instructor or the person next to you for help.

## Configuring the Matrix server

The installation process requires some basic config.

You will be asked to provide a hostname for your server.
If you didn't configure your own hostname earlier, use the hostname from the handouts. (e.g. `server01.chat.cryptoa.us`)

If asked for reporting anonymous stats, choose ‘no’. Nobody wants that.

Then, start your server:

```
systemctl start matrix-synapse
```

## Adding encryption support

It's time to encrypt this sucker.

Use certbot to generate a Let's Encrypt certificate. Read more about it here if you're curious: https://certbot.eff.org/

Run: `certbot certonly`

Choose the "spin up a temporary webserver" option.

And, so you don't have to worry about this every three months or whatever, we can set up certificate auto-renewal.

Run:
`crontab -e`

Insert the following line:
`@daily certbot renew --quiet --post-hook "systemctl reload nginx"`

### Configuring nginx

To make this thing truly HTTPS-ready, we need to configure a reverse proxy. We'll use nginx for this, so install it:

`apt-get install nginx -y`

Then add the following configuration to `/etc/nginx/conf.d/matrix.conf`:

```
server {
    listen 443 ssl;
    server_name matrix.chat.cryptoa.us;

    ssl_certificate     /etc/letsencrypt/live/matrix.chat.cryptoa.us/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.chat.cryptoa.us/privkey.pem;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;

    location /_matrix {
        proxy_pass http://localhost:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

Make sure you replace `matrix.chat.cryptoa.us` with the relevant server name.

Once that's saved, restart nginx by running:
`systemctl restart nginx`

## Fine-tuning Synapse

Add a shared secret to the config file at `/etc/matrix-synapse/homeserver.yaml`:

`registration_shared_secret: <add random characters here, whatever you want your secret to be>`

In that same config file, we also need to comment out the following:

```
#trusted_third_party_id_servers:
#    - matrix.org
#    - vector.im
```

Synapse caches conversation information in RAM where possible, and will use as much as you give it. For small implementations, (>50 users), you probably only need about 512MB of RAM. You can configure this by adding an environment variable to the following file:
`/etc/default/matrix-synapse`:

`SYNAPSE_CACHE_FACTOR 0.02`

And to make sure it all takes, restart the service:

`systemctl restart matrix-synapse`

### Register the first Matrix user

One of the things you probably want out of this chat server is to, y'know, chat with people.
To do that, we need some user accounts, starting with your own.

Create a new user by running the following, and answering the prompts:

`register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml https://localhost`

```
- New user localpart [root]: {add your name/handle here}
- Password:
- Confirm password:
- Make admin [no]: yes
- Sending registration request…
- Success.
```

**Optional:** to save having to register new users via CLI on your server every time, you can enable GUI user registration through the Riot client by editing `/etc/matrix-synapse/homeserver.yaml` and changing the following setting:

`enable_registration: true`

Otherwise, to register additional users, run `register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml https://localhost` again to manually configure more accounts.

Don't make them all admins, yeah?

## Time to Riot

Riot is the fancypants front-end client for the server we just set up.

If you don't have it already, you can download the app for your OS of choice at https://riot.im/

One you have it, run it.

Riot may try to auto-connect you to their default servers. If this happens, log out. We want the Riot login screen for the next part.

Let's connect Riot to the server we just configured.
Add your hostname (either your BYO hostname, or the here's-what-we-prepared-earlier hostname on your handout):

Home server URL: (e.g. `server01.chat.cryptoa.us`)

Identity server URL: (e.g. `server01.chat.cryptoa.us`)

Now log in with the user you configured in your server in the previous section of this doc.

### Create a new secure room

- Click on the gear icon and turn on end-to-end encryption by ticking ‘Enable encryption’
- Click ‘Save’

### Security checkup

- Who can access this room? -> Only people who have been invited (default)
- Who can read history? -> Members only (since they joined)
- URL previews -> Disable URL previews
- To invite users in the room -> Moderator

### More config

There's a bunch of stuff you can do with the Riot client. Explore the interface.

Here are some things to try:

- Invite your friends to the room you just created
  - compare key fingerprints before chatting in encrypted rooms
- Have someone else create a room (or prevent someone else from creating a room)
- Integrate Giphy for maximum lulz
- Add a GitHub bot

## Additional things to do after the workshop

There's server config we skipped over because this is a pretty short workshop.

We'd recommend going back to your server and doing some of these things later if you actually want to use Matrix/Riot properly.

- Deploy a firewall with iptables
- Configure auto-update (unattended-upgrades)
- Protect your Debian server with two-factor authentication
- Replace sqlite database with psql (if you expect lots of users): https://github.com/matrix-org/synapse/blob/master/docs/postgres.rst
- Configure email notifications (`enable_notifs`) - beware, may leak sensitive data!
- Add room integrations
  - GitHub bot
  - RSS bot
  - Giphy
- Add TURN support for audio/video calls

You can find all of the docs you could ever need (and the Matrix community itself) right here: https://matrix.org/docs/guides/faq.html#servers

## Post-event infra

To set up a Matrix server on your own infrastructure, you will need to provide your own domain name and your own server. We used AWS EC2 **t2.micro** instances as the servers for our workshop today - you may need something bigger if you have a large organisation to host.

After you buy a domain, configure your DNS settings as outlined in the "Add DNS records" section above, and then add these into your server's config in place of our temporary ones in the guide. Don't forget to generate another SSL cert through Let's Encrypt for your new hostname.

Make sure to restart Matrix after you're done making config changes, or they won't work properly.

Happy chatting!

## Interested in learning more?

Why not join join one of our Meetup groups nearby? We organise hands-on workshops like this on a regular basis.

Check out Meetup groups:

- [Hack for Privacy](https://www.meetup.com/hack-for-privacy/)
- [CryptoAUSTRALIA](https://www.meetup.com/pro/cryptoaustralia/)
