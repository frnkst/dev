# The fabulous world of pac files

## Prelude

This is the story of how I fell into the rabbit whole of proxy automatic configuration (or short PAC) files. My company lets me connect to their network by vpn. But we also use cloud services like Github. The enterprise Github is secured by a ip allow list, so it can only be accessed from legitimate devices. In our case that traffic coming from the outgoing forward proxy. The vpn somehow doesn't route all traffic through that proxy, so this needs to be configured on the client machine. So far so good, let's get started.

## The simple solution

On macOS you can go to WI-FI Settings > Your wifi name > Proxies. Lets set the HTTP and HTTPS proxy to "proxy.company.ch" and the port to 8080. Now I can access Github without any issues. I can work and everything is fine, until I want to google something. Wait what? Google.com gives me an error. And so does pretty much the rest of the internet. I get a ERR_TUNNEL_CONNECTION_FAILED. Looks like the vpn can't resolve any other hostnames than github.com and a few other ones. Probably this was configured by hand or there is no proper dns resolver configured? I'll probably never know. If there would only be a way to tell the system to use the proxy only for certain domains. Enter the world of PAC files.

## The PAC file

Luckily the company is hosting a PAC file. The Proxy auto-config file defined how web browsers and other user agents can automatically choose the appropriate proxy server for fetching a given URL. The file contains the JavaScript function `FindProxyForUrl(url, host)`, which returns a string with one or more access methods.

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

Cool. Send traffic to github.com and other important domains through the proxy and everything else not.

Let's just use that file. Go to WI-FI Settings > Your wifi name > Proxies. Turn automatic proxy configuration on and set the URL to https://company.com/proxy.pac or whatever the link to the pac file is. As soon as you click ok the file is loaded from the server. However, in the browser everything is like before. That's because the browser loads the pac file on startup. So let's close the browser and open it again. Cool Github.com works and so does the rest of the internet. All fixed, let's get to work. I use IntelliJ and there is this neat feature to see Github Pull Requests in the IDE. But wait, it's not working. Let's go to the IntelliJ Settings > Proxy and activate "Auto-detect proxy settings". It still doesn't work. Maybe I also need to specify the PAC file location explicitly: Click on "Automatic proxy configuration URL" and set the URL there. Maybe the IDE just needs a restart. Still nothing. What domains does the PR plugin connect to? I start a local intercepting proxy and point IntelliJ there, so I can analyse the traffic. It sends some POST requests to https://api.gitbhub.com/graphql.

Isn't that strange? `api.github.com/graphql` should definitely match `*github.com`. Let's verify this online [with a pac file tester](https://thorsen.pm/proxyforurl). The url `http://api.github.com/graphql` returns PROXY and so does `http://github.com`. Let's try it also [on this page](https://pactester.online/) for good measure. Same. How come IntelliJ is not sending this traffic to the proxy, but the browser is?

I fire up a transparent proxy, so I can do a man-in-the-middle and analyze the requests and responses in detail: `mitmproxy -p 9001 --mode upstream:http://company.proxy.com:8080`. There are a couple of requests being made, all of them go to api.github.com. And guess what: The feature itself now works in IntelliJ. So it's not really the proxy behaving weird and it must have something to do with the pac handling in IntelliJ. So I create my own pac file I can play around with.

## What the PAC!

Let's try to find a pac configuration that works for my use-case. There are rumors that you can use a local pac file, so I add `file:///Users/myuser/proxy.pac` in the wifi settings and make a simple config to redirect all traffic to mitmproxy:

```js
function FindProxyForURL(url, host) {
  return "PROXY localhost:9001";
}
```

That doesn't so anything. My experience is the same as these guys had [here](https://serverfault.com/questions/957519/why-does-file-users-username-library-proxy-pac-not-work-in-macos). And [this discussion](https://discussions.apple.com/thread/251395256?sortBy=rank) never got an answer but 150 upvotes. Then I host my own pac file on a server.

## The hosted PAC

I use the same pac file and start a webserver in that directory: `python3 -m http.server 9001 --bind 127.0.0.1`

Then I set the PAC config to the hosted file: `http://localhost:9001/proxy.pac`. Now that I have a solution, that actually works, I can play around with the config.

I immediately notice that there are a bunch of requests as soon as I click ok:

![alt text](images/initial.png)

This makes sense, because it needs to fetch the new config. But there is also a request as soon as I open Chrome or IntelliJ. Why is that? I thought that the network interface somehow routes the traffic to the correct location, but that's not how it works. When I specify a pac file in the WIFI settings AND I configure a client application to use the system settings, then the client application gets, parses and evaluates the pac file itself.

A pac file has been invented by Netscape in 1996, so it's been around for a long, long time. It's just a JavaScript file, with a bunch of helper or utility functions that are available. A pac file contains the one function `FindProxyForURL`. Inside the function the helpers like `shExpMatch` and `isInNet` are available. For parsing a JS file you need a JavaScript Engine. Luckily the browser already has a JS engine. It fetches the file, parses it, evaluates the code and applies to rules for redirecting the traffic. But where do those helper function come from? Well, they are baked in the browser. They are only available in a small sandbox called `pac-sandbox` inside the JS engine. This is also the place where the pac file is being executed.


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









