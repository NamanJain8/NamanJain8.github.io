---
layout: post
title: Postman Automation
---

I believe **“Any boring and repetitive task that can be automated should be automated”**. That comes with a pinch of salt (Reference: [https://xkcd.com/1205/](https://xkcd.com/1205/)).

<br><center><img src="/static/img/posts/postman-automation/automation_grid.png" alt="Automation Grid" width="90%"/><center><br>


We developers heavily use Postman or similar software for API development and testing. It is highly enriched with features like environments, variables, pre-request scripts, collection runner, etc. One of the critical features that we will be discussing is the **pre-request script**.

A pre-request script associated with a request will execute before the request is sent. It provides capabilities like dynamically setting data and variables, request manipulation, conditional logic, etc.

<img src="/static/img/posts/postman-automation/request_exec_order.png" alt="Request Execution Order" width="100%"/>

One of the major use case is setting the `Authorization` header in the request. This token is generally fetched from the remote server. Using pre-request script, this can be done easily.

```javascript
pm.sendRequest("https://auth.xyz.com/token", function (err, response) {
    pm.environment.set("Authorization", JSON.parse(response.text())["token"])
});
```

Let’s go over one of the use-case that I encountered recently. Consider the following setup simplified for the purpose of blog.
- Catalogue service contains the list of books read by a given user. Access to the catalogue of a given user requires the valid user token.
- We are a super-admin impersonating the user for testing the APIs. An “Auth CLI” exists to generate the impersonation token. Another team/third-party manages the CLI and the Auth Server. Hence, we don’t have direct access to the auth server APIs.

<br><center><img src="/static/img/posts/postman-automation/setup.svg" alt="Setup" width="80%"/><center>

In our setup, while testing, we might have to switch users, generate a token using CLI for each user, paste it into the Postman environment, and execute the API. The token would have some validity, after which it expires. We will have to regenerate the token and continue. This is boring and calls for automation. We could have completely relied on Postman's pre-request script, but it runs in a sandboxed environment and has a few limitations.

- It _supports only a few limited external libraries_ that limit what can be done via this script. For example, we can’t access the filesystem to persist the responses.
- _CLI commands cannot be executed_ from the scripts. Not all external services provide API support, or they might not be well-documented APIs. Instead, they might just offer decent CLI access to do things.

Moreover, not everyone is comfortable with Javascript. Developers would prefer writing complex logic for scripting in the language of their choice.

**Introducing Token Service:** It is an intelligent service that generates the token for a given user, caches it, and refreshes the expired token on demand. It internally runs the Auth CLI as direct API access to the Auth Server is not available or straightforward for a variety of reasons.

With a pre-request script, we call Token Service, eliminating token generation hassle during user switches or expiration. We can also save responses to the filesystem for later analysis using a post-response script integrated with a local service that sanitizes the data before writing.

#### Why would we need to run such service locally?

- Entire development setup is local, and we don’t want to introduce any remote dependency.
- Authn/Authz details might be stored on the local device, and gets updated frequently for compliance reasons.
- Token service might be in development, and we want to iterate fast. Once developed completely, it could be hosted on the cloud.

### Token Service as LaunchAgent

We've automated token generation partially, but currently, we must manually launch this service locally to utilize Postman APIs. It could be more efficient to set it up as a launch agent that runs whenever the machine runs.

In MacOS, this can be done by configuring this service to run as **Launch Agent**. It can be done simply in a few steps.

1. Create a plist file `com.local.token.plist` under `~/Library/LaunchAgents/` directory. Here, we are configuring it to run at load time, and monitor using logs.

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

Now, the service can be managed using launchctl. For example, if we update the service definition or the binary, the service can be force restarted using `launchctl kickstart -kp com.local.token` command. In linux, service can be launched as daemon using [systemd](https://www.linode.com/docs/guides/start-service-at-boot/).

The complete code for the example is available on [GitHub](https://github.com/NamanJain8/postman-automation) with instructions for running included.
