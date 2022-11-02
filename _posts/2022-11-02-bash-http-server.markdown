---
layout: post
title:  "A simple HTTP server in Bash"
tags: bash script shell linux network HTTP TCP
---

I work on a distributed system featuring micro-services. Each service produces events published to a number of subscribers through [webhooks](https://en.wikipedia.org/wiki/Webhook). Subscribers interested in the events of a service send a POST HTTP request to an endpoint with a notification URL in the body encoded in JSON. The service registers this address and sends POST requests to it when its internal state change. The application has a web interface but it is sometimes interesting to probe a specific service to see what it's publishing in real time. 

In this article I'll demonstrate a simple way to dump all this information in a terminal as the service changes state using [netcat](https://en.wikipedia.org/wiki/Netcat) and Bash. 

## Using netcat as a TCP server

First things first, here is the command to bind to a TCP socket and dump the incoming data on the standard output: 
```bash
nc -l -p 9876
```
`-l` stands for "listen" and `-p` for "port", obviously the port may be anything. Some versions of netcat allow you to give the port to `-l` directly (`nc -l <port>`). With this, you may send a request with curl (in a different shell) and see the result in the terminal where netcat is running:

```bash
nc -l -p 9876

# In a different terminal
curl localhost:9876 -d 'Hello World!'

# You should see in the first terminal something like: 
POST / HTTP/1.1
Host: localhost:9876
User-Agent: curl/7.75.0
Accept: */*
Content-Length: 12
Content-Type: application/x-www-form-urlencoded

Hello World!
```

If you try this on you machine, you'll notice that both applications hang until curl times out (use curl's `--max-time` to ) or you interrupt either of the processes. This is because curl sends an HTTP request while netcat is simply listening on a TCP connection. HTTP is typically sent over TCP so both programs can connect but curl expects an answer that never comes. On the other side netcat listen untils the TCP connection is closed by the client (curl). As long as curl doesn't receive its response (or the timeout delay expires), it keeps the TCP connection alive and both programs seem to be hanging. This is not a problem with curl as it's only doing what the HTTP protocol mandates.  
To solve this issue, we'll need netcat to send an HTTP response to curl so that it may close the connection.

## Responding to the HTTP request

When prompted to listened to a TCP connection, netcat will send whatever is on its standard input to the socket as it connects. Try for example: 

```bash
nc -l -p 9876 <<< "Hello back to you!"

# In a different terminal
curl localhost:9876 -d 'Hello World!'
> curl: (1) Received HTTP/0.9 when not allowed
```

curl received something, but that's not valid HTTP 1 so it prints an error and terminates. The TCP connection is closed and netcat should close too.

HTTP is basically just formatted text, so in order to have the connection close cleanly without an error message from curl, we can simply write our response in a file and feed it to netcat.

**response.http**
```http
HTTP/1.1 204 OK
Content-Length: 0

```

This is the most basic HTTP response you can get. A header indicating we're using HTTP 1.1, a response code (204 meaning success with no content. i.e. no body), and a _Content-Length_ of 0 indicating that there is nothing in our body. The extra empty line separates the header from the body of the response (which is empty in this case). There's a bunch of things that can go in the header but for this demonstration this is enough.  
Now: 
```bash
nc -l -p 9876 </path/to/response.http

# In a different terminal
curl localhost:9876 -d 'Hello World!'

# Both processes should exit cleanly at this point
```

## A simple webhook server

With this we have all we need. We can simply put our call to netcat in a loop in a script, register an url in the service we want to monitor and print whatever the it is sending our way. Of course, we probably don't care about the HTTP header, so we'll _sed_ it away. I use the fantastic [jq](https://stedolan.github.io/jq/) to do all the JSON processing and formatting. I strongly recommend it if you have to manipulate JSON, it's a game-changer. 

```bash
#!/usr/bin/env bash

# For simplicity's sake I skip over all the error handling that _is_ necessary in such a script
netcat_port="$1"
service_url="$2"

# Assuming the service expects a "subscription-url" field to contain the address where to send its data and
# responds with an "unsubscription-url" telling us where to send a DELETE request to unsubscribe
unsubscription_link="$(curl "$service_url" -Ss -X POST -H 'Content-Type: application/json' -d '{"subscription-url": "localhost:'"$port"'"}' | jq '.["unsubscription-url"]')"

function unsubscribe() {
  keep_running=false
  curl "$unsubscription_link" -Ss -X DELETE -H 'Content-Type: application/json' 
}

# This will guarantee that we're running the unsubscription procedure when we hit Ctrl+C to exit the script
trap unsubscribe EXIT

keep_running="true"
while [[ "$keep_running" == true ]]; do 
  # The sed command will deletes everything from the first line to the first empty line, inclusive
  nc -l -p "$netcat_port" </path/to/response.http | sed '1,/^$/d' | jq '.' # process the events as needed here
done 
```

A lot simpler than writing a dedicated application, isn't it? Obviously a complete script handling all possible errors is a lot longer and harder to write, but this is a good starting point if you need a quick-and-dirty solution to receive and print incomming HTTP requests. 
