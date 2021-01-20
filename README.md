# renku-vnc

## 1. Introduction

WORK IN PROGRESS.

Our objective is to run an X11 desktop and applications inside a jupyterlab based docker container, and display them in an HTML canvas present through our jupyterlab interface or full screen. Because the container is deployed and managed by renkulab, whatever method we use must piggy back on the existing jupyter connection.

A VNC-based approach with a javascript RFB to HTML5 renderer seems to be the easiest solution.

We try and compare the methods below.

In all configurations we use [novnc](https://github.com/novnc/noVNC) in order to render the RFB events into HTML on the browser side. All are served behind a [Jupyter server proxy](https://github.com/jupyterhub/jupyter-server-proxy) inside a [Docker](http://docker.com) container, except for the nginx approach which is used only for debugging purpose.

----
**1. Jupyter server proxy - novnc+websockify - x11vnc - xvbf**

![ws-x11vnc-xvfb](/svg/ws-x11vnc-xvfb-diag.svg)

* **Advantages**
  - It just works
  - Xvbf built on native OS distributor's X.org/X11 implementation, for better compatibility

* **Disadvantages**
  - overhead of x11vnc on top of xvbuf (to be confirmed in this exercise)
  - More proxies than required? websockify may not be necessary if libvncserver, on which x11vnc is built, supports websockets

----
**2. Jupyter server proxy - novnc - x11vnc - xvbf**

![x11vnc-xvfb](/svg/x11vnc-xvfb-diag.svg)

* **Advantages**

  - It is supposed to work
  - Xvbf built on native OS distributor's X.org/X11 implementation, for better compatibility
  - Take advantage of libvncserver's support for websockets and get away without the need for websockify

* **Disadvantages**
  - overhead of x11vnc on top of xvbf (to be confirmed in this exercise)

* **Caveats**
  - Libvncserver theoretically supports websocket, but [see novnc's issue 1310](https://github.com/novnc/noVNC/issues/1310)), and all related issues.

----
**3. Jupyter server proxy - novnc+websockify - TigerVNC (Xvnc)**

![ws-xvnc](/svg/ws-xvnc-diag.svg)

* **Advantages**
  - TigerVNC (Xvnc) built on native OS distributor's X.org/X11 implementation, for better compatibility
  - Remove overhead of separate x virtual frame buffer xvfb and vnc server, TigerVNC combines both in one

* **Disavantages**
  - No support for websockets? can't get away without overhead of intermediate websockify

----
**4. In-house solution based on customised versions of above component, optimised for our use case**

After investigating all the above configurations, we may find out that we prefer XRDP after all, or that we are better off forking our own VNC server implementation or novnc-like solution that is cusomized for our needs.

----
**5. Nginx**

![nginx-vnc](/svg/nginx-vnc-diag.svg)

We include this nginx configuration for debugging purpose only. So that we can verify the Docker container with the VNC stack outside the renkulab and jupyterlab context.

## 2. Implementations details

TODO - break down section below in separate folders

### 2.1 Nginx

Can be RUN in Dockerfile, e.g. from a renku base image without running jupyterlab.

This configuration is only for debugging purpose, in order to serve the novnc js code and proxify the websocket connection through a same port.

* [Install nginx](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/) as root or sudoer
```
release=$(grep -Po '(?<=VERSION_CODENAME=).*' /etc/os-release)
echo "deb https://nginx.org/packages/ubuntu/ ${release:-bionic} nginx
deb-src https://nginx.org/packages/ubuntu/ ${release:-bionic} nginx
" > /etc/apt/sources.list.d/nginx.list
sudo apt-get update
sudo apt-get install nginx
```

If _If a W: GPG error: https://nginx.org/packages/ubuntu xenial Release: The following signatures couldn't be verified because the public key is not available: NO PUBKEY $key_:

Take note of $key, and replacing $key by its proper value, run
```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys $key
sudo apt-get update
sudo apt-get install nginx
```

* Create a simple novnc virtual host in `/var/www/noVNC/` with the novnc resources as root or sudoer
```
cd /var/www
git clone https://github.com/novnc/noVNC.git
cd noVNC
rm -rf .git
```

* Configure nginx http context for websocket, as root or sudoer, e.g. create an ws.conf file
```
"${EDITOR:-vi}" /etc/nginx/conf.d/ws.conf
```

And copy the text below
```
##
# Websockets
##
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

* Create a virtual host
```
"${EDITOR:-vi}" /etc/nginx/sites-enabled/vnc
```

And copy the text below
```
server {
    listen 8888;
    listen [::]:8888;

    error_log /tmp/nginx.tmp warn;

    server_name jupyter.renkulab.io;

    root /var/www/noVNC;

    location / {
        index vnc_lite.html;
        try_files $uri $uri/ =404;
    }

    location /ws/ {
        proxy_pass http://localhost:5901;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_set_header Host $host;
    }
}
```

* Build the image, and start your container with `docker run --rm -it -p 8888:8888 image-name /bin/bash`

* Run `service nginx restart` inside the container to start the ngnix service

You can now access the vnc page on `localhost:8888/vnc.html?path=/ws/`. There must be a VNC service (RFB protocol) with websocket support listening on 5901. See the methods below in order to run the service (without the jupyter server proxy).

* **Notes**:
    - Port 8888 is used to connect from the host computer to nginx listening on port 8888 inside the docker container
    - Inside the container, nginx maps / locations to files under /var/www/noVNC, and streams the /ws/ location to localhost:5901
    - A VNC service supporting the websocket RFB protocol must listen on port 5901 inside the container.
    
* **Notes**:
    - This is WIP - novnc's RFB client can connect to x11vnc service which dumps protocol handshake logs that appear normal. However connection hangs and times out - it could be related to the libvncserver bugs mentioned above (there are several references to it). Updating libvncserver as recommended as a quick fix did not seem to help. This is under investigation while I am reviewing the logs.

### 2.2 Jupyter server proxy - novnc+websockify - x11vnc - xvbuf

Can be RUN in Dockerfile.

* Install x11vnc, Xvbf, and a lightweight Window or Desktop Manager (e.g. openbox, fluxbox, lxde/lxqt or xfce4)

Note: we are only aiming for a functional windows or desktop environment that can be used for testing purpose,
therefore this install lacks the obvious bells and whistles customizations.

```
apt-get update \
    && apt-get install -y --no-install-recommends \
       xvfb \
       x11vnc \
       net-tools \
       openbox \
       xterm \
    && apt-get autoremove --purge \
    && apt-get clean
```

* Install novnc and its sister project websockify
```
cd /var/www
git clone https://github.com/novnc/noVNC.git
rm -rf noVNC/.git
git clone https://github.com/novnc/websockify.git /var/www/noVNC/utils/
rm -rf noVNC/utils/websockify/.git
```

* Install and activate the [Jupyter server proxy](https://github.com/jupyterhub/jupyter-server-proxy) extension
```
conda install jupyter-server-proxy -c conda-forge
jupyter labextension install @jupyterlab/server-proxy
```

* Optionally create startup scripts for xvfb, x11vnc and novnc

This can be invoked by /entrypoint.sh, possibly via supervisord or can be done manually for testing purposes.
Keep in mind however that we don't have root/sudo priviledges in containers started by renkulab. Our startup script must work as a powerless $NB_USER userid.

```
FD_OPTS="-nolisten tcp -c r" FD_GEOM=1024x768x24 FD_PROG=/usr/bin/openbox x11vnc -display WAIT:cmd=FINDCREATEDISPLAY-Xvfb -noipv6 -noxrecord -rfbport 5901 -ping 5 -forever -nopw
/var/www/websockify/
/var/www/noVNC/utils/launch.sh --listen 6081 --vnc localhost:5901
```

* Build, and run the image in a container
```
docker run --rm -ti -p 8888:8888 <image> jupyter lab --ip=0.0.0.0
```

* Open a browser at the specified URL (with the token), and replace the URL's path by `/lab/proxy/6081/vnc.html?path=/lab/proxy/6081/`

