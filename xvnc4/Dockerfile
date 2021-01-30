#ARG RENKU_BASE_IMAGE=renku/renkulab-py:3.7-0.7.2
ARG RENKU_BASE_IMAGE=renku/renkulab-py:3.7-9d3bfa3
FROM ${RENKU_BASE_IMAGE}

#################################################################
USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        vnc4server \
        x11-xserver-utils \
        xfonts-base \
        xterm \
        net-tools \
    && mkdir -p /usr/share/novnc \
    && git clone https://github.com/novnc/noVNC /tmp/novnc \
    && mv /tmp/novnc/app /tmp/novnc/utils /tmp/novnc/core /tmp/novnc/vendor /tmp/novnc/package.json /tmp/novnc/*.html /usr/share/novnc \
    && mkdir -p /usr/share/novnc/utils/websockify \
    && git clone https://github.com/novnc/websockify /tmp/ws \
    && mv /tmp/ws/websockify /tmp/ws/run /tmp/ws/websockify.py /usr/share/novnc/utils/websockify \
    && sed -i -e 's,"websockify","vnc/",g' /usr/share/novnc/vnc.html \
    && sed -i -e "s,'websockify','vnc/',g" /usr/share/novnc/app/ui.js \
    && sed -i -e "s,'websockify','vnc/',g" /usr/share/novnc/vnc_lite.html \
    && ln -s /usr/share/novnc/vnc.html /usr/share/novnc/index.html \
    && chmod a+rX -R /usr/share/novnc \
    && apt-get autoremove --purge \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/* \
    && find /var/log -type f -exec cp /dev/null \{\} \;

COPY --chown=root:root startvnc /

#################################################################
# Custom X-session/window manager - pick at least one,
# and comment out what you don't want.
# Modify startvnc as needed so that it uses the selected WM.
##

#----------------------------------------------------------------
# openbox
#----------------------------------------------------------------
# RUN apt-get install -y \
#         openbox \
#     && apt-get autoremove --purge \
#     && apt-get clean \
#     && rm -rf /var/lib/apt/lists/* \
#     && rm -rf /tmp/* \
#     && find /var/log -type f -exec cp /dev/null \{\} \;

#----------------------------------------------------------------
# fluxbox
#----------------------------------------------------------------
RUN apt-get update && apt-get install -y \
        fluxbox \
    && apt-get autoremove --purge \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/* \
    && find /var/log -type f -exec cp /dev/null \{\} \;

#----------------------------------------------------------------
# xfce4
#----------------------------------------------------------------
# RUN apt-get install -y \
#         xfce4 \
#     && apt-get autoremove --purge \
#     && apt-get clean \
#     && rm -rf /var/lib/apt/lists/* \
#     && rm -rf /tmp/* \
#     && find /var/log -type f -exec cp /dev/null \{\} \;

# Install the jupyter extensions
USER ${NB_USER}

RUN conda install jupyter-server-proxy numpy -c conda-forge \
    && jupyter labextension install @jupyterlab/server-proxy \
    && conda clean -y --all \
    && echo "c.ServerProxy.servers = {\n\
    'vnc': {\n\
        'command': ['/startvnc', '{port}'],\n\
        'timeout' : 10,\n\
        'absolute_url': False,\n\
        'new_browser_tab': True,\n\
        'launcher_entry' :  {\n\
            'enabled': True,\n\
            'title': 'VNC'\n\
        }\n\
    }\n\
}" >> ~/.jupyter/jupyter_notebook_config.py



# Install the python dependencies
USER ${NB_USER}

COPY --chown=${NB_USER}:${NB_USER} requirements.txt environment.yml /tmp/
RUN conda env update -q -f /tmp/environment.yml && \
    /opt/conda/bin/pip install -q -r /tmp/requirements.txt && \
    rm -f /tmp/requirements.txt -f /tmp/environment.yml && \
    conda clean -y --all && \
    conda env export -n "root"

# RENKU_VERSION determines the version of the renku CLI
# that will be used in this image. To find the latest version,
# visit https://pypi.org/project/renku/#history.
ARG RENKU_VERSION=0.12.0

########################################################
# Do not edit this section and do not add anything below

USER ${NB_USER}

RUN if [ -n "$RENKU_VERSION" ] ; then \
    pipx uninstall renku && \
    pipx install --force renku==${RENKU_VERSION} \
    ; fi

########################################################