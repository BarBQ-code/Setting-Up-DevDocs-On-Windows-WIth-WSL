# Setting Up DevDocs On Windows With WSL

So I wanted to setup devdocs on my local machine for offline documentation so I followed the Quick Start Documentation documented on their [github](https://github.com/freeCodeCamp/devdocs) and it all worked fine. The devdocs server is listening on port 9292 and you have to access it with through localhost, which is kinda ugly, I wanted to browse to [devdocs.my].

So I found a way using `netsh` to port forward connections locally which looks something like this:

```batch
netsh interface portproxy add v4tov4 listenport=80 listenaddress=127.0.92.92 connectport=9292 connectaddress=127.0.0.1
```

Combining that with adding a host in the hosts file that looks like this:

`127.0.92.92    devdocs.my`

Theoretically (and practically) I should be able to browse to [devdocs.my], and it worked, except I got a weird error message.

[Here is the full StackOverflow thread that talks about that](https://stackoverflow.com/questions/8652948/using-port-number-in-windows-host-file)

The error message stated that cookies should be enable or DevDocs will not work properly, and it didn't work as expected.
So I started googling and I didn't find an answer, at first I thought it had something to do with the Host header, and maybe DevDocs checks if the header equals to localhost/127.0.0.1, but I couldn't find any mention of that. I tried a lot of different stuff until I found out that modern browsers block cookies with hostnames that don't have certificates. So here is a full tutorial of how to setup DevDocs.

## Prerequisites
WSL - You can install it on windows without it, but we are going to use nginx and it's easier to setup in linux. You can also follow this tutorial on a Linux machine.

Ruby => 2.7.3 - DevDocs is built on top of Ruby. (You can install this version using rbenv)

Docker - DevDocs comes with a docker file which is easy to use, you can set it up without it, and it's documented in their [github](https://github.com/freeCodeCamp/devdocs) But it's easier with docker.


### Setting up docker
Start by cloning their repo and building and running their docker image
```sh
git clone https://github.com/freeCodeCamp/devdocs.git && cd devdocs

docker build -t thibaut/devdocs .
docker run --name devdocs -d -p 9292:9292 thibaut/devdocs
```
Now you should be able to browse to your devdocs locally through port 9292.

### Adding a hostname
Add to your hosts file the hostname you wish to browse to to acccess devdocs.

It should look like this:

`127.0.0.1  <host>`

### Creating Certificates
To create certificates we are going to use [mkcert](https://github.com/FiloSottile/mkcert).

To install `mkcert` we first need to install `certutil`:
```sh
sudo apt install libnss3-tools
```

Then we can install it using [brew](https://brew.sh/).
```sh
brew install mkcert
```

Now we can generate certificates using `mkcert`. Run these commands:
```sh
mkcert -install
mkcert <host> # the same host you configured in your hosts file
```

Now two files should be created in your current directory, a certificate key and the certificate itself.

### Setting up nginx
We are going to use nginx as a reverse proxy, and nginx will serve our certificates.

Install nginx:
```sh
sudo apt install nginx
```

Copy this config file to `/etc/nginx/sites-enabled/default`:
```nginx
server {
    listen *:443 ssl;
    server_name <host>;

    ssl_certificate <PATH-TO-YOUR-CERTIFICATE.pem>;
    ssl_certificate_key <PATH-TO-YOUR-CERTIFICATE-KEY.pem>

    location / {
        proxy_pass http://localhost:9292;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        include /etc/nginx/proxy_params;
        proxy_redirect off;
    }
}

server {
    if ($host = <host>) {
        return 301 https://$host$request_uri;
    }

    listen 80;
    server_name <host>;
    return 404;
}
```
This configuration sets up two server listeners, one listening on port `443`, this server will setup an SSL connection and redirect every request to localhost:9292, the second server listens on port `80` and checks if the request comming in is for your host, if it is, it will redirect the traffic to the first server, if it's not it will return `404` Not found.

The only things you need to change in this configuration are in these <> brackets.

Now you should be able to browse to your host. (Chrome and other browsers might warn you that it's not a safe connection but you can just ignore it).