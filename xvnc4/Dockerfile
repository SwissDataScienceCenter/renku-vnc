# For finding latest versions of the base image see
# https://github.com/SwissDataScienceCenter/renkulab-docker
ARG RENKU_BASE_IMAGE=renku/renkulab-py:3.7-0.7.3
FROM ${RENKU_BASE_IMAGE}

#################################################################
USER root

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        vnc4server \
        xfonts-base \
        x11-xserver-utils \
        xterm \
        net-tools \
    && mkdir -p /usr/share/novnc \
    && git clone https://github.com/novnc/noVNC /tmp/novnc \
    && mv /tmp/novnc/app /tmp/novnc/utils /tmp/novnc/core /tmp/novnc/vendor /tmp/novnc/package.json /tmp/novnc/*.html /usr/share/novnc \
    && mkdir -p /usr/share/novnc/utils/websockify \
    && git clone https://github.com/novnc/websockify /tmp/ws \
    && mv /tmp/ws/websockify /tmp/ws/run /tmp/ws/websockify.py /usr/share/novnc/utils/websockify \
    && sed -i -e "s,'websockify',window.location.pathname.slice(1),g" /usr/share/novnc/app/ui.js \
    && sed -i -e "s,'websockify',window.location.pathname.slice(1),g" /usr/share/novnc/vnc_lite.html \
    && ln -fs /usr/share/novnc/vnc_renku.html /usr/share/novnc/index.html \
    && chmod a+rX -R /usr/share/novnc \
    && apt-get autoremove --purge \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/* \
    && find /var/log -type f -exec cp /dev/null \{\} \;

COPY --chown=root:root vnc_renku.html /usr/share/novnc/
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

# install the jupyter extensions
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



# install the python dependencies

COPY requirements.txt environment.yml /tmp/
RUN conda env update -q -f /tmp/environment.yml && \
    /opt/conda/bin/pip install -q -r /tmp/requirements.txt && \
    rm -f /tmp/requirements.txt -f /tmp/environment.yml && \
    conda clean -y --all && \
    conda env export -n "root"

# RENKU_VERSION determines the version of the renku CLI
# that will be used in this image. To find the latest version,
# visit https://pypi.org/project/renku/#history.
ARG RENKU_VERSION=0.13.0

########################################################
# Do not edit this section and do not add anything below

RUN if [ -n "$RENKU_VERSION" ] ; then \
    pipx uninstall renku && \
    pipx install --force renku==${RENKU_VERSION} \
    ; fi

########################################################
