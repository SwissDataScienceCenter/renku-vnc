# RENKU-VNC

## 1 Jupyter Server Proxy, NoVNC, Websockify, x11vnc, Xvfb or Xdummy

#### 1.1 Usage

* Build the image in this folder: `docker build -t renku-vnc .`

* Start the docker container: `docker run --rm -it -p 8888:8888 renku-vnc jupyter lab --ip=0.0.0.0`

* Open the jupyter lab page at the URL provided in the output, or copy the token `http://127.0.0.1:8888/?token=...`

* Open the HTML vnc viewer on `localhost:8888/vnc`

#### 1.2 Implementation details

* Install X virtual frame buffer (Xvfb), the VNC server (x11vnc), and net-tools (netstat)

```
apt-get update
apt-get install -y --no-install-recommends xterm xvfb x11vnc net-tools
```

* Install the window manager, and X utilities (xset). We use fluxbox, but other choices such as openbox, xfce4 are also possible.

```
apt-get update
apt-get install -y fluxbox x11-xserver-utils
```

Note, we are only aiming for a functional windows or desktop environment that can be used for testing purpose,
therefore this install lacks the obvious bells and whistles customizations. See the documentation on the available windows manager or full
fledge desktop managers for more information.

* Install novnc and its sister project websockify from github

```
mkdir -p /usr/share/novnc
git clone https://github.com/novnc/noVNC /tmp/novnc
mv /tmp/novnc/{app,utils,core,vendor,package.json} /tmp/novnc/*.html /usr/share/novnc
mkdir -p /usr/share/novnc/utils/websockify
git clone https://github.com/novnc/websockify /tmp/ws
mv /tmp/ws/{websockify,run,websockify.py} /usr/share/novnc/utils/websockify
chmod a+rX -R /usr/share/novnc
```

* Make `vnc/` the default path for NoVNC's websockify. Note that the '/' is required. Using `websockify` instead of `vnc` in the c.ServerProxy.servers configuration below to avoid this modification would not work.

```
sed -i -e 's,"websockify","vnc/",g' /usr/share/novnc/vnc.html
sed -i -e "s,'websockify','vnc/',g" /usr/share/novnc/app/ui.js
sed -i -e "s,'websockify','vnc/',g" /usr/share/novnc/vnc_lite.html
```
* Soft-link vnc.html to index.html so that we land on vnc.html from the jupyter lab's VNC launcher

```
ln -s /usr/share/novnc/vnc.html /usr/share/novnc/index.html
```

* Install and activate the [Jupyter server proxy](https://github.com/jupyterhub/jupyter-server-proxy) extension

```
conda install jupyter-server-proxy -c conda-forge
jupyter labextension install @jupyterlab/server-proxy
```

* Copy and paste the startup scripts into `/starvnc` to start xvfb, x11vnc and novnc:

```
(x11vnc -env FD_OPTS="-nolisten tcp -c r" \
            -env FD_GEOM="1024x768x24" \
            -env FD_PROG="/usr/bin/startfluxbox" \
            -display WAIT:cmd=FINDCREATEDISPLAY-Xvfb \
            -no6 \
            -noipv6 -noxdamage -noxfixes -noxrecord \
            -rfbport ${vncport} \
            -ping 5 \
            -repeat \
            -forever -nevershared -nopw -o /tmp/x11vnc.log & ) &
(/usr/share/novnc/utils/launch.sh --vnc localhost:${vncport} --listen ${wsport} > /tmp/novnc.log 2>&1 & ) &
```

The x11vnc server will automatically look for an available display, and start Xvfb to create one if none exists (-display argument).
The environment variables `FD_OPTS` and `FD_GEOM` can be used to pass additional arguments to the Xvfb process.

Modify `FD_PROG` to start the window manager of your choice.

X11vnc is configured to work in an endless loop (`-forever`) and allow a single client at the time (`-nevershared`), disconnecting
existing clients when a new client (websockify proxy) connects.
All communications between the jupyter lab proxy and Xvfb are unencrypted and password-less (`-nopw`) since they all happen inside the
user's container and are protected to the jupyter lab proxy endpoint.

See the [Xvfb](https://linux.die.net/man/1/xvfb) and [x11vnc](https://linux.die.net/man/1/x11vnc) documentation for other tuning parameters.

Keep in mind however that we don't have root/sudo priviledges in containers started by renkulab. Our startup script must work as a powerless $NB_USER userid.

* Configure the jupyter lab server proxy extension to automatically invoke the `/startvnc` script when accessing the `vnc/` path on the jupyter lab's URL.

In `~/.jupyter/jupyter_notebook_config.py` copy paste the lines:

```
c.ServerProxy.servers = {
     'vnc': {
         'command': ['/startvnc', '{port}'],
         'timeout' : 10,
         'absolute_url': False,
         'new_browser_tab': True,
         'launcher_entry' :  {
            'enabled': True,
            'title': 'VNC'
     }
}
```

Jupyter proxy server will specify the `{port}` number. This is the port on which the VNC server or novnc's websockify proxy must listen to.

It is possible to also specify an icon for the VNC launcher in `launcher_entry`.
See the [Jupyter server proxy documentation](https://jupyter-server-proxy.readthedocs.io/en/latest/server-process.html) for details.



