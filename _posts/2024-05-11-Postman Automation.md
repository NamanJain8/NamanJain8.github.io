---
layout: post
title: Postman Automation
---

I believe **“Anything that can be automated, should be automated”**. That comes with a pinch of salt.

<img src="/static/img/posts/postman-automation/automation_grid.png" alt="Automation Grid" width="100%"/>

Reference: [https://xkcd.com/1205/](https://xkcd.com/1205/)

As a developer, we heavily use Postman or similar software for API development and testing. It provides highly enriched support like environments, variables, pre-request scripts, collection runner, etc. One of the important feature that we will be discussing is pre-request script.

A pre-request script associated with a request will execute before the request is sent. It provides capabilities like dynamically setting data and variables, request manipulation, conditional logic, etc.

<img src="/static/img/posts/postman-automation/request_exec_order.png" alt="Request Execution Order" width="100%"/>

One of the major use case is setting the Authorization header in the request. This token is generally fetched from the remote server. Using pre-request script, this can be done easily.

```javascript
pm.sendRequest("https://auth.xyz.com/token", function (err, response) {
    console.log(response.json());
    pm.environment.set("Authorization", response.json()["token"])
});
```

Let’s go over one of the use-case that I encountered recently. Consider the following setup simplified for the purpose of blog.
- Catalogue service contains the list of books read by a given user. Access to the catalogue of a given user requires the valid user token.
- We are a super-admin testing the APIs and have to impersonate as user. There exists an “Auth CLI” to generate the impersonation token. The CLI and the Auth Server is managed by some other team or third-party. Hence, we don’t have direct access to the auth server APIs.

<img src="/static/img/posts/postman-automation/setup.svg" alt="Setup" width="100%"/>

As Postman runs within a sandboxed environment, there are few limitations of the scripting.
- Postman supports only few limited external libraries that limits what can be done via this script. For example, we can’t access the filesystem to persist the responses.
- CLI commands cannot be executed from the scripts. Not all the external services might provide API support, or it might not be well documented APIs. Rather they might just provide a decent CLI access to do things.

Moreover, not everyone is comfortable with Javascript. Developers would prefer writing complex logic for scripting in the language of their choice.

In our setup, while testing we might have to switch from user to user; generating token using CLI for each of the user, paste it in Postman environment and execute the API. The token would have some validity after which it expires. We will have to regenerate the token and continue. This is boring and calls for automation.

**Introducing Token Service:** It is a smart service that generates the token for a given user, caches it and refreshes the expired token on demand. It internally runs the Auth CLI as direct API access to Auth Server is not be available or straightforward for variety of reasons.

Using pre-request script, we can call Token Service and we are free from hassle of generating token as we switch from user to user users or the token expires.

Similarily, we can persist the response into filesystem for analysis later. This can be done using the post-response script integrated with a service running locally. This local service would write to filesystem after sanitising the response.

#### Why would we need to run such service locally?

- Entire development setup is local, and we don’t want to introduce any remote dependency.
- Authn/Authz details might be stored on the local device, and gets updated frequently for compliance reasons.
- Token service itself might be in development phase; and we want to iterate fast. Once developed completely, it could be hosted on cloud.

### Token Service as LaunchAgent

We have automated the small part of token generation but still we have to launch this service locally everytime we want to use these Postman APIs. It might be better to spin this up as a launch agent, that runs whenever the machine is running. 

In MacOS, this can be done by setting this service as Launch Agent. It can be done simply in few steps.

1. Create a plist file `com.local.token.plist` under `~/Library/LaunchAgents/` directory. Here, we are adding the file to dump the logs, and setting it up to run at load time.

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
        <key>Label</key>
        <string>com.local.token</string>
        <key>ProgramArguments</key>
        <array>
            <string>/usr/local/bin/token-server</string>
        </array>
        <key>RunAtLoad</key>
        <true/>
        <key>StartInterval</key>
          <integer>1</integer>
        <key>StandardErrorPath</key>
        <string>/tmp/token.err.log</string>
        <key>StandardOutPath</key>
          <string>/tmp/token.out.log</string>
    </dict>
    </plist>
    ```
2. Load the launchd service definition file (plist) and start

    ```bash
    launchctl load ~/Library/LaunchAgents/com.local.token.plist
    ```

Now, the service can be managed using launchctl. For example, if you update the service definition or the binary, the service can be force restarted using `launchctl kickstart -kp com.local.token` command.

The same can be done via [systemd in Linux](https://www.linode.com/docs/guides/start-service-at-boot/).

Entire code for this sample can be accessed at [Github Repo](https://github.com/NamanJain8/postman-automation).
