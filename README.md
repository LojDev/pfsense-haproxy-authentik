This documentation aims to outline the steps I did to get Authentik working via HAProxy on pfSense. 

## Before We Begin
I've tried my best to make this documentation as user and noob friendly as possible so even a complete beginner will be able to follow and get everything working. This documentation will not touch on every single thing but I did try to explain even the thing that I thought was common knowledge, so to all the seasoned persons reading, just bear with me... I want this to be noob friendly.

### Some assumptions:
1. You will be using docker to host Authentik and you already have docker / docker compose installed in your environment. I will not touch on this as it is very easy to install and there are lots of tutorials on the internet about it, plus installing docker may vary depending on your OS and environment so for those reasons I will not touch on that.
2. I will be writing this documentation with the assumption that everyone following along has a "clean" environment, meaning they do not already have Authentik installed and do not have HAProxy installed and configured on pfSense. If you do not have a "clean" environment, you can still follow along but you will need to make necessary adjustments based on your environment.
3. It is assumed that you already have a pfSense instance, whether it be pfSense+ or pfSense CE, whether you have dedicated Netgate Hardware or running pfSense in a VM. Note: steps will be the same whether you are running pfSense+ or CE. 

### My Environment
The following network diagram is a snippet of my homelab. I will be using the information in the diagram to aid in this documentation.
![1.png](file:///C:%5CUsers%5Cl.buchanan%5CDropbox%5CObsidian%5CImages%5Cnotes%5Cpfsense%5Cauthentik-haproxy%5C1.png)

## Step 1 - Download Required Files
This repo contains a copy of my docker compose stack I used to setup my Authentik instance. It also contain the lua files that are needed to be imported on pfSense to get HAProxy working. I also included links to to original repos in the table below where I got these files from in case you may want to get them from the source.

| Name                 | Type          | content                                                                                                                                                                |
| :------------------- | :------------ | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| json.lua             | Lua script    | copied from [https://github.com/rxi/json.lua](https://github.com/rxi/json.lua)                                                                                         |
| haproxy-lua-http.lua | Write to disk | copied from [https://github.com/haproxytech/haproxy-lua-http](https://github.com/haproxytech/haproxy-lua-http)                                                         |
| auth-request.lua     | Lua script    | copied from [https://github.com/TimWolla/haproxy-auth-request/blob/main/auth-request.lua](https://github.com/TimWolla/haproxy-auth-request/blob/main/auth-request.lua) |

## Step 2 - Setup Authentik

To get started logon to your docker host and download the official docker compose file using the following command
```bash
wget https://goauthentik.io/docker-compose.yml
```

You will need to create a `media`, `certs`, and `custom-templates` folder to hold your Authentik data.
```bash
mkdir media certs custom-templates
```

You will now need to generate a secret key for Authentik and a password for postgres and store them in a `.env` file. To do so run the following commands
```bash
echo "PG_PASS=$(openssl rand -base64 36 | tr -d '\n')" >> .env
echo "AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')" >> .env
```

Now you just need to run the containers. Do so by running the following command
```bash
docker compose up -d
```

After Authentik is running, browse to `https://[ip-of-docker-host]:[authentik-https-port]/if/flow/initial-setup/` (In my case it would be `https://192.168.50.10:9444/if/flow/initial-setup/`). You should be presented with a page to setup user email and password.

### Configure Authentik Outpost, Provider and Application
Now that you setup Authentik and is logged in we will need to setup a Provider and an Application to handle Forward Auth requests.

The first is to setup a new Provider for Forward Auth (Domain Level). In Authentik go to `Applications -> Providers` and create a new Provider. Select `Proxy Provider` and click Next to continue.
![2.png](file:///C:%5CUsers%5Cl.buchanan%5CDropbox%5CObsidian%5CImages%5Cnotes%5Cpfsense%5Cauthentik-haproxy%5C2.png)

On the next screen give the provider a name and set Authentication Flow to ***...explicit-consent..***. Make sure `Forward auth (domain level)` is selected and enter the Authentication URL and Cookie domain.

The `Authentication URL` is the proxy URL that you will use to access Authentik via HAProxy and the `Cookie domain` is your root domain.
![3.png](file:///C:%5CUsers%5Cl.buchanan%5CDropbox%5CObsidian%5CImages%5Cnotes%5Cpfsense%5Cauthentik-haproxy%5C3.png)

Now scroll down to the `Advanced flow settings` section and make sure `Authentication flow` and `Invalidation flow` are set. I am using the defaults here.
![4.png](file:///C:%5CUsers%5Cl.buchanan%5CDropbox%5CObsidian%5CImages%5Cnotes%5Cpfsense%5Cauthentik-haproxy%5C4.png)

Next is to setup a new Application. In Authentik go to `Applications -> Applications` and create a new Application.

Give the application a name and slug and ***make sure to set the provider for the application to the one you just created*** 
![5.png](file:///C:%5CUsers%5Cl.buchanan%5CDropbox%5CObsidian%5CImages%5Cnotes%5Cpfsense%5Cauthentik-haproxy%5C5.png)

Next is to set the embedded outpost to use the Forward Auth Provider you created. In Authentik go to `Applications -> Outposts` and you should see the default embedded outpost
![6.png](file:///C:%5CUsers%5Cl.buchanan%5CDropbox%5CObsidian%5CImages%5Cnotes%5Cpfsense%5Cauthentik-haproxy%5C6.png)

Edit the embedded outpost and make sure to copy the Forward Auth Application you created earlier over to the `Selected Applicaatios` box
![7.png](file:///C:%5CUsers%5Cl.buchanan%5CDropbox%5CObsidian%5CImages%5Cnotes%5Cpfsense%5Cauthentik-haproxy%5C7.png)

Expand the `Advanced settings` section and for `authentik_host` you want to put the prosy URL that you will use to access Authentik over HAProxy. This should be the same URL you entered as the Authentication URL in the provider.
![8.png](file:///C:%5CUsers%5Cl.buchanan%5CDropbox%5CObsidian%5CImages%5Cnotes%5Cpfsense%5Cauthentik-haproxy%5C8.png)

And that should be it for the setting up your Authentik instance. We will now move over to pfSense and getting HAProxy setup in the coming steps.


## Step 3 - Install and configure HAproxy

On pfSense go to `System -> Package Manager -> Available Packages` and Search for haproxy. You will most likely see 2 haproxy packages, a `haproxy-devel` and the standard `haproxy` package. Install the standard haproxy package 

### Uploading lua files to pfSense
To Import the files on pfSense, go to `Diagnostics -> Command Prompt` and use the `Upload File` section to upload the 3 lua files. The files will be uploaded to the `/tmp/` directory on pfSense.

Now you need to copy/move the files to the `/usr/local/share/lua/5.3` folder so HAProxy will be able to load those files. Again, on the `Diagnostics -> Command Prompt` screen, you need to use the `Execute Shell Command` section to copy the files from the `/tmp/` directory to the `/usr/local/share/lua/5.3` directory. Do so by executing the following shell commands:

```bash
cp /tmp/json.lua /usr/local/share/lua/5.3/json.lua
```

```bash
cp /tmp/haproxy-lua-http.lua /usr/local/share/lua/5.3/haproxy-lua-http.lua
```

```bash
cp /tmp/auth-request.lua /usr/local/share/lua/5.3/auth-request.lua
```

### Configure HAProxy to use the lua files
Go to `Services -> HAProxy` and under the `Files` tab add the 3 lua files. When adding the files, you will need to copy all the contents of the files you are adding and paste it in the content section on pfSense. Use the image below as reference:
![9.jpg](file:///C:%5CUsers%5Cl.buchanan%5CDropbox%5CObsidian%5CImages%5Cnotes%5Cpfsense%5Cauthentik-haproxy%5C9.jpg)

After adding the files, save and apply config, you should NOT get any errors. If you get any errors or warnings then something went wrong so recheck everything you did up to this point.

## Step 4 - Configure Backends

After successfully adding the required files from [[pfSense-HaProxy-Authentik Setup#Step 3 - Install and configure HAproxy|Step 3]] above, you can start to configure the backends.

### Setup Backend for Authentik
You need to setup a backends for Authentik. To do so, go to `Services -> HAProxy -> Backend` and add a backend.

For the backend I will simply name mine as `authentik-http`, fell free to name your backend whatever you want. This name will be relevant later on in the tutorial so remember it. You will also need to populate the Address and Port fields with the IP address of your Authentik instance and the port for http, then save. See image below for reference
![Screenshot 2025-05-01 101410.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20101410.png)

### Setup backend you want to protect
In addition to setting up the backend for Authentik, you will also need to setup the backend/s for any service/s you want to protect. 

For the sake of this tutorial lets say I have a service (snipe-it) that I want to protect. This service is running at 192.168.12.3 on http port 4000.

The backend setup is the same, give it a name and populate the Address and Port.
![Screenshot 2025-05-01 104631.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20104631.png)

Now just save and apply config. You should NOT get any errors. If you get any errors or warnings then something went wrong so recheck everything you did up to this point.


## Step 5 - Configure Frontend

### Create Frontend And Set Listen IP and Port
Now you need to setup a frontend to tie everything together. To do so, go to `Services -> HAProxy -> Frontend` and add a frontend.

You can name the frontend whatever you want, the important thing is to set the Listen Address you want HAProxy to listen on and also the port you want it to listen on. See image below for reference:
![Screenshot 2025-05-01 110159.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20110159.png)

### Setting up Access Control Lists
In the front end you will need to setup the ACL below ensuring it is the first ACL in the list. This ACL will determine which backends are protected by authentik based on the url.
```
acl protected-frontends hdr(host) -m reg -i ^(?i)(service1|service2)\.yourdomain\.com
```
Based on the ACL above the backends at `https://service1.yourdomain.com` and `https://service2.yourdomain.com` will be protected by a authentik login prompt. You can add more services by just adding a `|` and adding the other service you want to protect. For example if you want to protect a service at `https://service3.yourdomain.com` then the ACL would be modified to be `(service1|service2|service3)\.lojlocal\.com`

See images below for reference:
![Screenshot 2025-05-01 112007.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20112007.png)

Now setup the following ACLs as well.

```
acl protected-frontends var(txn.txnhost) -m reg -i ^(?i)(service1|service2)\.yourdomain\.com
```
Reference:
![Screenshot 2025-05-01 112657.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20112657.png)


```
acl is_authentikoutpost path -m reg ^/outpost.goauthentik.io/
```
Reference:
![Screenshot 2025-05-01 112708.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20112708.png)


```
acl is_authentikoutpost var(txn.txnhost) -m reg ^/outpost.goauthentik.io/
```
Reference:
![Screenshot 2025-05-01 112715.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20112715.png)


For each backend you want to protect you will need to add an ACL for it, specifying the fqdn that backend service should be proxied through. Example below

```
acl service1 hdr(host) -i service1.yourdomain.com
```
Reference:
![Screenshot 2025-05-01 112723.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20112723.png)


You will also need to create an ACL for authentik itself also and this one is important because whatever fqdn you use here should match the one set in Authentik. For mine, I have it set to `authentik.mydomain.com`.

![Screenshot 2025-05-01 112805.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20112805.png)



When you are done setting up the ACLs, you should have something similar to mine shown below:
![Screenshot 2025-05-01 114359.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20114359.png)


### Setting up Actions
The actions controls what happens when a incoming request to the proxy matches any ACLs.

```
http-request set-var(req.scheme) str(https) if { ssl_fc }
```
If this is not set then there will be a warning in Authentik that HTTPS is not detected correctly. From my very short testing I did not see any noticeable impact to the functionality of Authentik when that warning was displayed, but I think it's worth pointing this out.


```
http-request set-var(req.scheme) str(http) if !{ ssl_fc }
```
Reference:
![[Screenshot 2025-05-01 115645.png]]



```
http-request set-var(req.questionmark) str(?) if { query -m found }
```
Reference:
![[Screenshot 2025-05-01 115807.png]]



```
http-request set-header X-Real-IP %[src]
```
Reference:
![[Screenshot 2025-05-01 120840.png]]

![[Screenshot 2025-05-01 120906.png]]



```
http-request set-header X-Forwarded-Method %[method]
```
Reference:
![[Screenshot 2025-05-01 121015.png]]

![[Screenshot 2025-05-01 121030.png]]



```
http-request set-header X-Forwarded-Proto %[var(req.scheme)]
```
Reference;
![[Screenshot 2025-05-01 121056.png]]

![[Screenshot 2025-05-01 121109.png]]



```
http-request set-header X-Forwarded-Host %[req.hdr(Host)]
```
Reference:![[Screenshot 2025-05-01 121204.png]]

![[Screenshot 2025-05-01 121224.png]]



```
http-request set-header X-Original-URL %[url]
```
Reference:
![[Screenshot 2025-05-01 121342.png]]

![[Screenshot 2025-05-01 121358.png]]



```
http-request lua.auth-intercept authentik-http_ipvANY /outpost.goauthentik.io/auth/nginx HEAD x-original-url,x-real-ip,x-forwarded-host,x-forwarded-proto,user-agent,cookie,accept,x-forwarded-method x-authentik-username,x-authentik-uid,x-authentik-email,x-authentik-name,x-authentik-groups - if protected-frontends !is_authentikoutpost
```
Reference:
![Screenshot 2025-05-01 121457_2.png](file:///C:%5CUsers%5Cl.buchanan%5CDownloads%5CScreenshot%202025-05-01%20121457_2.png)

Make note of the `authentik-http_ipvANY`. This is the authentik http backed created earlier in the documentation, pfSense automatically adds `_ipvANY` to all backend names in the haproxy config. You should replace `authentik-http` with the name of your backend.


```
http-request redirect code 302 location /outpost.goauthentik.io/start?rd=%[hdr(X-Original-URL)] if protected-frontends !{ var(txn.auth_response_successful) -m bool } { var(txn.auth_response_code) -m int 401 } !is_authentikoutpost
```
Reference:
![[Screenshot 2025-05-01 122821.png]]

![[Screenshot 2025-05-01 122839.png]]


```
http-request deny if protected-frontends !{ var(txn.auth_response_successful) -m bool } { var(txn.auth_response_code) -m int 403 } !is_authentikoutpost
```
Reference:
![[Screenshot 2025-05-01 122907.png]]

![[Screenshot 2025-05-01 122956.png]]


```
http-request redirect location %[var(txn.auth_response_location)] if protected-frontends !{ var(txn.auth_response_successful) -m bool } !is_authentikoutpost
```
Reference:
![[Screenshot 2025-05-01 123019.png]]

![[Screenshot 2025-05-01 123035.png]]



```
http-response set-header Strict-Transport-Security "max-age=63072000"
```
Reference:
![Screenshot 2025-05-01 123126.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20123126.png)



```
http-request set-var(txn.txnhost) hdr(host)
```
Reference:
![Screenshot 2025-05-01 123226.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20123226.png)



```
http-request set-var(txn.txnpath) path
```
Reference:
![Screenshot 2025-05-01 123241.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20123241.png)



```
use_backend authentik-http_ipvANY if protected-frontends is_authentikoutpost
```
Reference:
![Screenshot 2025-05-01 125120.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20125120.png)


Now you should add actions for all the backends. 
![[Screenshot 2025-05-01 125508.png]]

![[Screenshot 2025-05-01 125538.png]]
***NOTE:*** The names in the `See below` field should match the the names used when setting up the ACLs. So the names above should be the same as the name used when creating the ACLs below:
![Screenshot 2025-05-01 130019.png](file:///C:%5CUsers%5Cl.buchanan%5CPictures%5CScreenshots%5CScreenshot%202025-05-01%20130019.png)

## Step 6 - Testing

That should be it. Now you just need to test. Browse to the URL of any of your services that you are protecting with the `protected-frontends` ACL and it should redirect you to Authentik for authentication before you get access to the service.