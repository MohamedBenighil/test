# Working with docker behind a proxy

In order to build and run the container, you need to configure the proxy address or URL :

1. DNS IP address in the docker daemon, so it can find the proxy if you configured an URL
2. Host OS, so docker service can pull the images
3. Docker image environment, so you can use apt-get (cf. [Configure Docker to use a proxy server](https://docs.docker.com/network/proxy/))

## 1. Configure DNS at daemon start

I first tried to put it in dns.conf under /etc/systemd/system/docker.service.d/ following the syntax as for http-proxy.conf as it is explained in answers from stackoverflow (cf. [DNS for docker with systemd](https://stackoverflow.com/questions/33784295/setting-dns-for-docker-daemon-on-os-with-systemd#33804503)):

```bash
[Service]
ExecStart=
ExecStart=/usr/bin/docker daemon --dns 10.192.65.254 --dns 10.192.64.254 -H fd://
```

but it only made docker daemon unable to start up. I suppose I did not write the command properly or I am duplicating things.

So I commented all the lines and the docker daemon came back.

Then I found the best solution in the second answer from [askubuntu.com](https://askubuntu.com/questions/475764/docker-io-dns-doesnt-work-its-trying-to-use-8-8-8-8#476955)

Simple and effective:

- Edit/Create /etc/docker/daemon.json
- Type:

    ```JSON
    {
        "dns":["10.192.65.254", "10.192.64.254", "8.8.8.8"]
    }
    ```

For info:

- Orange DNS IPs:

  - 10.192.65.254
  - 10.192.64.254

- Google DNS:
  - 8.8.8.8

---

## 2. Proxy in host for docker service

### 2.1 Docker service

  The docker service needs to know the proxy URL/address in order to get internet access
  (cf. Docker documentation [HTTP/HTTPS proxy](https://docs.docker.com/config/daemon/systemd/#httphttps-proxy))

  The proxy has to be indicated in the directory "/etc/systemd/system/docker.service.d/"

  In that directory, edit/create the file "http-proxy.conf" with the content:

```bash
[Service]
Environment="HTTP_PROXY=http://proxy.rd.francetelecom.fr:8080/"
Environment="HTTPS_PROXY=http://proxy.rd.francetelecom.fr:8080/" "NO_PROXY=localhost,172.17.0.0/16, 172.18.0.0/16"
```

  After modifying files in docker.service.d, always:

- Flush changes:

```bash
sudo systemctl daemon-reload
```

- and restart Docker:

```bash
sudo systemctl restart docker
```

  Even if the Host OS does not have the proxy configurated, the docker service will arrive to get out IF we configure the environment in the previous file.

  If the proxy is not configured in docker.service.d, docker service will not arrive to get out even if the host OS has a proxy configuration.

### 2.2 System global configuration

  Ubuntu stores the system proxy configuration in the file: /etc/environment

  This is good to know in case the docker image must be configured as the host system.
  
  The typical use case: a new internal shell script in the docker image needs to reach internet because there is a call to apt-get, e.g. [mininet installation script](https://github.com/mininet/mininet/blob/master/util/install.sh)

  The environment variables wil not be passed to the new shell unless they are stored in a common place to all the shells: /etc/environment

## 3. Docker image environment

Once we start to build a docker image, we are INSIDE the docker image and no more in the host. So, the previous configuration is useless and the build will not success if we do not pass the proxy configuration to the image environment, cf. [Configure Docker to use a proxy server](https://docs.docker.com/network/proxy/)

We have three ways to reach our goal:

1. Put the ENV variables in the Dockerfile (less configurable)

    ```dockerfile
    ENV http_proxy "http://proxy.rd.francetelecom.fr:8080/"
    ENV https_proxy "http://proxy.rd.francetelecom.fr:8080/"
    ```

    Notice that the environment variables are **lowerCamelCase**.

2. Create/edit a JSON file in the .docker directory under the user's home: ~/.docker/config.json

    ```JSON
    {
    "proxies":
    {
      "default":
      {
        "httpProxy": "http://proxy.rd.francetelecom.fr:8080",
        "httpsProxy": "http://proxy.rd.francetelecom.fr:8080",
        "noProxy": "*.test.example.com,.example2.com"
      }
    }
    }
    ```

    Notice that the JSON keys are **lowerCamelCase**

3. Pass the proxy values as arguments of the "--build-args" option of docker build

    ```shell
    docker build . --build-args http_proxy=$http_proxy  --build_args https_proxy=$https_proxy -t awesome_image:1.0
    ```

  The second option should be prefered.

  It is more configurable and has the benefit to keep a single, or very little modified,version of Dockerfile regardless the need or not of a proxy.

  The third one should be used only for special cases when working in temporary develoment environment.

## Enjoy!

---

### (Update Aug, 16th 2018)

## Docker-compose

The developper can specify in the docker-compose.yml file to build the docker image when starting the services.
Though, the behaviour of docker-compose may raise issues if internal shell scripts inside the docker image need to reach the Internet as it is the case with the [mininet installation script](https://github.com/mininet/mininet/blob/master/util/install.sh).

When building with docker-compose the mininet-docker image behind a proxy, there is an error coming from git because of the use of gnutls_handshake() :
_gnutls_handshake() failed: An unexpected TLS packet was received._

The best way to bypass this issue is to build previously the image with "docker build" and then refer to the created image in the "image" key of docker-compose.yml

---

### Author: Juan GASCON (juan.gascon@orange.com)

Please, feel free to send me your modifications and/or suggestions.
