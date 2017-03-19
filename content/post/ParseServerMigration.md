+++
author = "kandelvijaya"
date = "2017-03-19T23:02:15+01:00"
description = "Running self hosted Parse Server with dashboard."
tags = []
title = "Backend As Service from Scratch"

+++

Parse was a great initiative. Sadly it went down. It was a fantastic solution for those who want to quickly
get their app done and backend was just a breeze. Things changed. However, not all is lost. Parse became open
source. The source for Server and Dashboard are both out in the open. I had a mobile app that used Parse heavily
. Parse shutdown was really hard for me. I went through all the alternatives but none got me more interested than
actually running the infrastructure on my own dedicated linux server.

This blog points out various reference which helped along the way, various setbacks(which became experience) and
tips that will help you guide through.

The end result, I moved everything to my hosted parse server and the app works just as it did before. Rest assure,
its pretty easy.

## Prerequisites (My way)
1. DigitalOcean 10$ instance
    - _[DigitalOcean](https://www.digitalocean.com)_ provides linux instance with various configuration. They have sensible defaults and extremely easy way of setting up server or remote VPS.  Moreover, the documentation, they are detailed and covers the whole spectrum of things you want/wish/need to do.

    - _I was a AWS user. Their humungous options drove me nuts to find the right thing(Paradox of Choice).  Plus, after 1 free year, it costs a lot._
2. Be able to __ssh__ into remote machine via terminal
    - _[How to configure SSH connection](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-freebsd-server)_
3. Install `nodejs`. This will install `npm` with it.
    - [To install Nodejs](https://www.digitalocean.com/community/tutorials/how-to-install-node-js-on-ubuntu-16-04)
    - [Nodejs](https://nodejs.org/en/) is a javascript runtime. It uses V8 Javascript engine and adds server specific features via
    C++ and are exposed to JS. V8 is open source and used in Chrome and Chromium browser.  
4. Install [Mongodb](https://www.mongodb.com): NoSQL database which Parse Server prefers.
    - [Here is the installation guide](https://www.digitalocean.com/community/tutorials/how-to-install-mongodb-on-ubuntu-16-04)
    - Its very __important to not use `mongodb-runner` npm package that Parse-Server example uses__. This particular package
    contains bunch of script to automate mongodb installation and running with a single command. The downside is __your database will be purged if the mongodb-runner is terminated or restarted__. This could happen for many reasons even in server environment.
5. Lets now get __Parse-Sever__
    - [Parse-Sever github Repo.](https://github.com/ParsePlatform/parse-server)
    - Issue this command to install it:

    ```
    $ npm install -g parse-server
    ```

    - This command is all you need to get and install __parse-server__.
    - __dont install mongodb-runner__. Mongodb-runner is for 1 time testing only.
6. If you want easy way to get started then, cloning this __parse-server-example__ is a valid option. [Check the details in the Readme](https://github.com/ParsePlatform/parse-server-example).
    - This is how I got my first parse server working in a maintainable way.
    - If you clone this repo, this will download __parse-server__ as a dependency; if not found or installed previously.
    - Modify your __index.js__ file to customize. Refer to the Readme if its little confusing. Don't panic, if you are afraid of javascript. We wont need much of it. Just some variable assignment.
7. Okay we have almost done everything besides 2 things.
    - We don't have dashboard so we cant see/edit/add new data to the backend.
    - We will do both in the next section. Now take a drink and lets get going.

## Intermediate Server work
Apache is a well known Web Server but we will be using NGINX on this particular case. You could go with Apache but I prefer NGINX as it is more memory effecient _(at least thats what I heard)_ and has nice configurations for us.

1. Install and configure [NGINX as described in this article](https://www.digitalocean.com/community/tutorials/how-to-install-nginx-on-ubuntu-16-04).
2. Open up port 4040 if you want to view the dashboard served from the server. _If you want the dashboard to be client side then thats totally fine._
    - `$ sudo ufw allow 4040/tcp`
3. Install __parse-dashboard__.
    - I prefer dashboard be in server. The reasons are 2. First, I can access server from any machine on the go. Second, it just a website interface to the parse-server. I would rather expose parse-dashboard than parse-server to the outside world.
    - install via  

    ```
    $npm install -g parse-dashboard
    ```

    - [Official parse-dashboard github repo](https://github.com/ParsePlatform/parse-dashboard)
    - `Readme` on the repo should help you get using the dashboard.
    - I prefer to have a `index.js` file with all the configuration wrapped in `express` app. This is more flexible than tedious command line options argument and json config based setup.
    - Again, Read the __readme__ very carefully. It helps a lot. It saves lots of googling for some minor things. Its all there. Waiting to be read. I thought I read all then, after some days, I kept googling how to pass 1 argument. Its not fun. Read the README.
4. Now, if you have followed through step by step. You should be able to run __parse-dashboard__ on port 4040. Remember we allowed 4040 to be reached from outside. Its time to check your dashboard.
    - From anywhere outside the server, browse to `<your-server-ip>:4040`
    - You should be able to see the dashboard with the parse-server app we created in the first place.
    - Get inside and add some data.
    - Notice, there is a section for __CloudCode__, __Push__, __WebHooks__ and __Logs__. Exactly, the open source community have grown to implement almost all that was in Parse.com
5. Now lets make our life simpler.

## Intermediate II Server Work
Lets assume, your Parse-Server is working fine. What if at somepoint in time, the system crashes due to many requests (usually a good sign to indicate your customer base has grown) or any other reason. What if parse-server crashes? Will you log in and restart yourself? I don't think so.

Enter __pm2__. A easy way to manage process. _We will use pm2 to restart parse-server if the existing instance crashes automatically_. This will also make sure parse-server is started on system reboot. Nice!!

1. Install [PM2](http://pm2.keymetrics.io)
    -
    ```
    $sudo npm install -g pm2
    ```

    - To make this _pm2_ package start on reboot or crash, you need to configure _Systemd_ or _Service_. [Look at PM2 section on this exhustive article](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-node-js-application-for-production-on-ubuntu-16-04)
    - Start the parse server using `$pm2 start <your-Parse-app-ServerPath>/index.js --name <yourappname>`
    - Start the parse dashboard using `$pm2 start <dashboardPath>/index.js --name <dashboardName>`
    - Now you should be able to see the running process via `$pm2 list`
    - To see logs for specific process, you can use `$pm2 logs <either app name or process id assigned by pm2>`. This will be helpful in debugging certain queries and cloud code or in general what the server is doing.

    ![PM2 list](/img/pm2list.png)

2. If you want to associate this IP to a domain name.
    - To get the domain name, I prefer [Namecheap](www.namecheap.com)
    - To configure DNS for the domain with common domain name registrars, [read this guide.](https://www.digitalocean.com/community/tutorials/how-to-point-to-digitalocean-nameservers-from-common-domain-registrars)
    - [How to configure to use domain name in the droplet](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-host-name-with-digitalocean)
    - To encrypt your domain name for free with TLS and have _https:_ [follow this guide](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-16-04).
        - Make sure to forward-proxy `ipaddress:4040` to `domainname.com/dashboardPath`.
        - Look at [Configuring HTTPS for NGINX section on this article](https://www.digitalocean.com/community/tutorials/how-to-set-up-a-node-js-application-for-production-on-ubuntu-16-04)
3. Daily MongoDB backup. Yes, you might need to protect your data after all.
    - Either choose to take snapshot every day and pay digitalocean some small amount.
    - Or, use cronjob to backup every day.
    - Backing up existing database is easy.
        - `$mongodump`
    - Restoring dumped data into database is easy too.
        - `$mongorestore <dumpfile>`
    - [This article will walk through making daily backups](https://coderwall.com/p/hdmmnq/easy-automatic-nightly-backups-for-mongodb)
4. Thats all. The rest will fall into place.


Hope you could manage to code your big app using a simple Parse backend.

As always, you can contribute to make this article more accurate or extend openly via [Github repo](https://github.com/kandelvijaya/com/tree/master/content/post)

### Cheers!
