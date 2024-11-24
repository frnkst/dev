# The fabulous world of pac files

## Prelude

This is the story of how I fell into the rabbit whole of proxy automatic configuration (or short PAC) files. My company lets me connect to their network by vpn. But we also use cloud services like Github. The enterprise Github is secured by a ip allow list, so it can only be accessed from legitimate devices. The vpn somehow doesn't route all traffic through their outgoing proxy server, so this needs to be configured in the system settings. So far so good. Let's get started:

## The simple solution

On macOS you can go to WI-FI Settings > Your wifi name > Proxies. Let set the HTTP and HTTPS proxy to "proxy.company.ch" and the port to 8080. Now we can access Github without any issues. I can work and everything is fine, until I want to google something. Wait what? Google.com gives me an error. And does pretty much the rest of the internet. ERR_TUNNEL_CONNECTION_FAILED. Looks like the vpn can't resolve any other hostnames then github.com and a few other ones. Probably this was configured by hand or there is some other DNS wizardry in place. Why can't they not just a standard dns resolver? We'll probably never know. If there would only be a way to tell the system to only use the proxy for certain domains. Enter the world of PAC files:

## The PAC file

Luckily the company is hosting a PAC file. The Proxy auto-config file defined how web browsers and other user agents can automatically choose the appropriate proxy server for fetching a given URL. The file contains the JavaScript function `FindProxyForUrl(url, host)`, which returns a string with one or ore access methods.

Let's look at the file that my company provides:

```js
function FindProxyForURL(url, host) {
  PROXY = "PROXY proxy.company.com:8080";
  
  if (shExpMatch(host, "*github.com|*.some-other-domain.com")) {
    return PROXY;
  }
  
  return "DIRECT";

}
```

Easy. Send traffic to *.github.com through the proxy and everything else not.

Let's just use that file. Go to WI-FI Settings > Your wifi name > Proxies. Turn automatic proxy configuration on and set the URL to https://company.com/proxy.pac or whatever the link to the pac file is. As soon as you click Ok the file is loaded from the server. However, in the browser everything is like before. That's because the browser loads the pac file on startup. So let's close the browser and open it again. Cool Github.com works and so does the rest of the internet. All fixed, let's get to work. I use IntelliJ and there is this neat feature to see Github Pull Requests in the IDE. But wait it's not working. Let's go to Settings > Proxy and set Auto-detect proxy settings. But it still doesn't work. Let's try to also set the PAC file here. Click on "Automatic proxy configuration URL" and set the URL there. Maybe the IDE just needs a restart. Still nothing. Let's see where this feature connects to. I start a local intercepting proxy and tell IntelliJ to connect to that proxy so I can see the traffic. It sends some POST requests to https://api.gitbhub.com/graphql.

Strange. `api.github.com/graphql` should definitely match `*.github.com`. Let's verify this online on https://thorsen.pm/proxyforurl. http://api.github.com/graphql returns PROXY and http://github.com returns PROXY. Let's try it also here for good measure: https://pactester.online/. Same. How come the connection from IntelliJ is sent via DIRECT. Let's also try it in the browser. Both URLs work.

There is a check connection button in IntelliJ. Let's see. Connection to https://api.github.com is fine. Connection to https://api.github.com/graphql is not! Makes sense because my request in not authenticated. So let's examine this with authenticated requests.

All this sounds like it had to lead to the use of mitmproxy. Well not really but it's just a fantastic tool for the job. Let's fire up mitmproxy to listen in and forward all traffic to the company's proxy: `mitmproxy -p 9001 --mode upstream:http://company.proxy.com:8080`. There are a couple of requests being made, all of them go to api.github.com. And guess what: The feature works in IntelliJ. So it's not really the proxy behaving weird, it's actually the PAC handling on macOS that is somehow broken. I'm currently on macOS Sonoma, for all the other version I don't know.

## What the PAC

Let's try to find a pac configuration that works for our use-case. There are rumors that you can use a local pac file. Let's try this by adding `file:///Users/myuser/proxy.pac` and make a simple config to redirect all traffic to mitmproxy so we can test if it works:

```js
function FindProxyForURL(url, host) {
  return "PROXY localhost:9001";
}
```

Nope, nothing. Those guys found out too: https://serverfault.com/questions/957519/why-does-file-users-username-library-proxy-pac-not-work-in-macos and https://discussions.apple.com/thread/251395256?sortBy=rank. There is no answer for 4 years. Let's host the file on a webserver so I can test it out.

## The hosted PAC

I use the same pac file and start a webserver in that directory: `python3 -m --bind localhost http.server`

Then I set the PAC config to the hosted file: `http://localhost:8080/proxy.pac`. Now that we have a solution that actually works we can play around with the config.

### First try: Redirect *.github.com to the proxy

```js
function FindProxyForURL(url, host) {
  PROXY = "PROXY localhost:9001";
  
  if (shExpMatch(host, "*.github.com")) {
    return PROXY;
  }
  
  return "DIRECT";

}
```

Pactester says it should redirect https://github.con and https://api.github.com  to the proxy. But it doesn't.

### Second try: Redirect *github.com  and *.github.com to the proxy

```js
function FindProxyForURL(url, host) {
  PROXY = "PROXY localhost:9001";
  
  if (shExpMatch(host, "*github.com | *.github.com")) {
    return PROXY;
  }
  
  return "DIRECT";

}
```

Pactester says it should redirect https://github.con and https://api.github.com  to the proxy. but it doesn't.

Desperation starts to kick in. Maybe it's how shExpMatch is being evaluated. Let's try it on a  separate line:

### Third try: On a separate line

```js
function FindProxyForURL(url, host) {
  PROXY = "PROXY localhost:9001";
  
  if(shExpMatch(host, "*.github.com")) {
    return PROXY;
  }
  
  if (shExpMatch(host, "*github.com")) {
    return PROXY;
  }
  
  return "DIRECT";

}
```

Pactester says it should redirect https://github.con and https://api.github.com  to the proxy. but it actually does. So we really did find a working solutions.

## The end

Cool, we have a working solution. What's not so cool is that there is no plausible explanation, why it behaves the way it does and not the way it actually should. But doing some research on PAC it seems like it's this fabulous, annoying world where you can never be sure what it does until you try it out. This quote from here seems to sum it up pretty well: https://www.reddit.com/r/sysadmin/comments/g78uhy/pulling_my_hair_out_with_pac_files/


> I have always struggled with this. It also doesn't help that certain browsers evaluate PAC files differently. Anyways, I've never found anything other than a syntax checker for PAC files. I've had to actually test them with live traffic to see how they behave.

## Continue here

- TODO  fix wording
- Add links where appropriate
- Add screenshots where neededed
- File IntelliJ bug report












## Or maybe it's just intelliJ?

Observation 1: 
- When I add the pac to wifi --> browser  respects it
- In IntlliJ
  --> Autodetect proxy settings --> nothing happends
  --> Autodetect and add pac file --> still doesn't work

Test it with the proxies






Check if it's BOM.
If the PAC file encoding is UTF-8 with BOM, it will not work. Make sure that the encoding is UTF-8 without BOM.

  

https://www.jetbrains.com/help/idea/settings-http-proxy.html


If the PAC file encoding is UTF-8 with BOM, it will not work. Make sure that the encoding is UTF-8 without BOM.









