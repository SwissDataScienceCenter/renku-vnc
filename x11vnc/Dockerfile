#ARG RENKU_BASE_IMAGE=renku/renkulab-py:3.7-0.7.2
ARG RENKU_BASE_IMAGE=renku/renkulab-py:3.7-9d3bfa3
FROM ${RENKU_BASE_IMAGE}

#################################################################
USER root

# Common
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        #openssl \
        #ca-certificates \
        xterm \
        xvfb \
        x11vnc \
        novnc \
        net-tools \
    && apt-get autoremove --purge \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/* \
    && find /var/log -type f -exec cp /dev/null \{\} \;

#################################################################
# Custom VNC - pick at least one,
# and comment out what you don't want
##
USER root

#################################################################
# Custom X-session/window manager - pick at least one,
# and comment out what you don't want.
# Modify configs as needed.
##

#----------------------------------------------------------------
# openbox (100MB)
#----------------------------------------------------------------
# RUN apt-get install -y \
#         openbox \
#     && apt-get autoremove --purge \
#     && apt-get clean \
#     && rm -rf /var/lib/apt/lists/* \
#     && rm -rf /tmp/* \
#     && find /var/log -type f -exec cp /dev/null \{\} \;

#----------------------------------------------------------------
# fluxbox (10MB)
#----------------------------------------------------------------
RUN apt-get update && apt-get install -y \
        fluxbox \
        x11-xserver-utils \
    && apt-get autoremove --purge \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* \
    && rm -rf /tmp/* \
    && find /var/log -type f -exec cp /dev/null \{\} \;

#----------------------------------------------------------------
# xfce4 (0.5GB)
#----------------------------------------------------------------
# RUN apt-get install -y \
#         xfce4 \
#     && apt-get autoremove --purge \
#     && apt-get clean \
#     && rm -rf /var/lib/apt/lists/* \
#     && rm -rf /tmp/* \
#     && find /var/log -type f -exec cp /dev/null \{\} \;


# Install the python dependencies
USER ${NB_USER}

COPY --chown=${NB_USER}:${NB_USER} requirements.txt environment.yml /tmp/
RUN conda env update -q -f /tmp/environment.yml && \
    /opt/conda/bin/pip install -q -r /tmp/requirements.txt && \
    conda install jupyter-server-proxy -c conda-forge && \
    jupyter labextension install @jupyterlab/server-proxy && \
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
