# renku-vnc

## 2 Jupyter Server Proxy, NoVNC, Websockify, Xvnc4

### 2.1 Usage

* Build the image in this folder: `docker build -t renku-vnc .`

* Start the docker container: `docker run --rm -it -p 8888:8888 renku-vnc jupyter lab --ip=0.0.0.0`

* Access the vnc page on `localhost:8888/vnc`, or via the jupyter lab VNC launcher.

### 2.2 Implementation details

* Install the VNC server (xvnc4), x terminal, xfonts-base, and net-tools (netstat)

```
apt-get update
apt-get install -y --no-install-recommends vnc4server x11-server-utils xterm xfonts-base net-tools
```

xfonts-base is a minimum recommendation so that Xvnc4 does not fail with the error

```
Fatal server error:
could not open default font 'fixed'
```

* Install the window manager, and X utilities (xrandr, xset). We use fluxbox, but other choices such as openbox, xfce4 are also possible.

```
apt-get update
apt-get install -y fluxbox x11-xserver-utils
```

Note that we are only aiming for a functional windows or desktop environment that can be used for testing purpose.
Therefore this install lacks the obvious bells and whistles customizations. See the documentation on the available windows manager or full
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

* Make `vnc/` the default path for NoVNC's websockify. Note that the '/' is required. Using `websockify` instead of `vnc` in the c.ServerProxy.servers configuration below to avoid this modification would not be sufficient.

```
sed -i -e 's,"websockify","vnc/",g' /usr/share/novnc/vnc.html
sed -i -e "s,'websockify','vnc/',g" /usr/share/novnc/app/ui.js
sed -i -e "s,'websockify','vnc/',g" /usr/share/novnc/vnc_lite.html
```
* Soft-link vnc.html to index.html so that we land on vnc.html from the jupyter lab's VNC launcher

```
ln -s /usr/share/novnc/vnc.html /usr/share/novnc/index.html \
```

* Install and activate the [Jupyter server proxy](https://github.com/jupyterhub/jupyter-server-proxy) extension

```
conda install jupyter-server-proxy -c conda-forge
jupyter labextension install @jupyterlab/server-proxy
```

* Copy the startup scripts into `/starvnc` to start xvfb, x11vnc and novnc:

```
(Xvnc4 :20
        -geom 1024x768 \
        -depth 24 \
        -c r \
        -rfbport ${vncport} \
        -SecurityTypes None \
        -xinerama \
        -localhost \
      > /tmp/xvnc4.log 2>&1 &) &
sleep 1
(/usr/share/novnc/utils/launch.sh --vnc localhost:${vncport} --listen ${wsport} > /tmp/novnc.log 2>&1 &) &
sleep 1
(/usr/bin/startfluxbox &) &
```

In the above, vncport is the RFB port on which the VNC server will be listening, and wsport is the port that the websockify proxy
will listen to for connections forwared by the jupyter lab webproxy. Wsport can be set by the jupyter lab proxy and passed as
an argument to the script (see the argument `{port}` below).

All communications between the jupyter lab proxy and Xvnc4 are unencrypted and password-less (`-SecurityTypes None`)
since they all happen inside the user's container, and external communications to the jupyter lab proxy endpoint are encrypted and protected by a token.

See the Xvnc4 documentation for other tuning parameters.

Keep in mind however that we don't have root/sudo priviledges in containers started by renkulab. Our startup script must work as a powerless $NB_USER userid.

* Configure the jupyter lab server proxy extension to automatically invoke the `/startvnc` script when accessing the `vnc/` path on the jupyter lab's URL.

In `~/.jupyter/jupyter_notebook_config.py` add the lines:

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

It is possible to also specify an icon for the VNC launcher in `launcher_entry`.
See the [Jupyter server proxy documentation](https://jupyter-server-proxy.readthedocs.io/en/latest/server-process.html) for details.


