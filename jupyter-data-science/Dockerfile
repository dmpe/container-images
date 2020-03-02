FROM fedora:latest
LABEL maintainer="dmpe <cincenko@outlook.com>"
ARG ALM_DOCKER_REG

ARG ALM_VCS_REF
LABEL org.label-schema.schema-version="2.0" \
      org.label-schema.name="Data Science Notebook - fedora" \
      org.label-schema.vendor="DevOps" \
      org.label-schema.description="datascience-notebook-fedora" \
      org.label-schema.version="2.0"

ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"

USER root

# Carefull: Oracle modifies URL and appends "x86_64" in the middle which is not visible on
# https://yum.oracle.com/repo/OracleLinux/OL7/oracle/instantclient/x86_64/index.html
RUN (set -e; \
    dnf upgrade -y; \
    dnf groupinstall -y "Development Tools" "Development Libraries"; \
    dnf install -y python3-pip libnsl libaio; \
    cd /tmp; \
    curl -k -LOJ http://yum.oracle.com/repo/OracleLinux/OL7/oracle/instantclient/x86_64/getPackage/oracle-instantclient19.5-basic-19.5.0.0.0-1.x86_64.rpm; \
    dnf install -y /tmp/oracle-instantclient19.5-basic-19.5.0.0.0-1.x86_64.rpm; \
  )

COPY fix-permissions /usr/local/bin/fix-permissions

RUN (set -e; \
    update-ca-trust; \
    dnf install -y R R-devel czmq-devel boost-devel QuantLib bash-completion python*-networkx tree mlocate jq bzip2 \
                   liberation-fonts dejavu-fonts-common icu4j-localespi wget octave octave-devel gnuplot \
                   graphviz-devel cairo-devel libjpeg-devel giflib-devel librsvg2-devel nodejs \
                   tig pandoc nano gfortran gcc-gfortran unixODBC-devel \
                   texlive-scheme-basic texlive-collection-latexextra; \
)

RUN (set -e; \
    python3 -m pip install numpy scipy ipython notebook jupyterhub jupyterlab pandas bqplot octave_kernel bash_kernel \
        beautifulsoup4 bokeh cloudpickle cython jupytext jupyterlab_sql psutil ipywidgets matplotlib \
        numexpr protobuf ipysheet fds.analyticsapi.engines nbresuse nbdime scikit-learn seaborn sqlalchemy \
        statsmodels vincent xlrd openpyxl xlsxwriter jupyter_bokeh jupyterlab-git ipympl cx_Oracle ;\
    python3 -m bash_kernel.install; \
    dnf clean all; \
)

# A must; sets yarn configuration; as a test check .yarnrc
RUN (set -e; \
    git config --system credential.helper store; \
    npm config set strict-ssl false; \
    npm install -g node-gyp-build node-gyp yarn; \
    echo "ulimit -c 0 > /dev/null 2>&1" > /etc/profile.d/disable-coredumps.sh;\
    yarn config set "strict-ssl" false; \
)

ENV NB_USER=$NB_USER \
    NB_UID=$NB_UID \
    NB_GID=$NB_GID \
    HOME=/home/$NB_USER \
    TINI_VERSION=v0.18.0 \
    LD_LIBRARY_PATH=/usr/lib/oracle/19.5/client64/lib:$LD_LIBRARY_PATH

# Copy a script that we will use to correct permissions after running certain commands
# Enable prompt color in the skeleton .bashrc before creating the default NB_USER
RUN chmod a+rx /usr/local/bin/fix-permissions && \
    sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc

# Create NB_USER wtih name jovyan user with UID=1000 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    useradd -m -s /bin/bash -N -u $NB_UID $NB_USER && \
    curl -LJ https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini --output /usr/local/bin/tini && \
    chmod +x /usr/local/bin/tini && \
    cd /tmp && \
    curl -LOJ https://github.com/PAIR-code/facets/archive/master.zip && \
    unzip facets-master.zip && cd facets-master && \
    jupyter nbextension install facets-dist/ --sys-prefix && \
    cd && rm -rf /tmp/facets && \
    fix-permissions $HOME

USER $NB_UID
WORKDIR $HOME
RUN mkdir /home/$NB_USER/work && \
    mkdir /home/$NB_USER/.jupyter && \
    echo "cacert=/etc/pki/tls/certs/ca-certificates.crt" > /home/$NB_USER/.curlrc && \
    chown -R $NB_USER:users /home/$NB_USER/.jupyter && \
    npm cache clean --force && \
    jupyter notebook --generate-config

# Copy local files as late as possible to avoid cache busting
COPY start.sh /usr/local/bin/
COPY start-notebook.sh /usr/local/bin/
COPY start-singleuser.sh /usr/local/bin/
COPY jupyter_notebook_config.py /etc/jupyter/

# Activate ipywidgets extension in the environment that runs the notebook server
USER root
RUN (set -e; \
    npm config set strict-ssl false; \
    yarn config set "strict-ssl" false; \
    jupyter nbextension enable --py widgetsnbextension --sys-prefix; \
    # Also activate ipywidgets extension for JupyterLab
    # Check this URL for most recent compatibilities
    # https://github.com/jupyter-widgets/ipywidgets/tree/master/packages/jupyterlab-manager
    jupyter labextension install @jupyter-widgets/jupyterlab-manager --no-build; \
    jupyter labextension install @bokeh/jupyter_bokeh --no-build; \
    jupyter labextension install @jupyterlab/git --no-build; \
    jupyter serverextension enable --py jupyterlab_git; \
    jupyter labextension install plotlywidget --no-build; \
    jupyter labextension install @jupyterlab/plotly-extension --no-build; \
    jupyter labextension install bqplot --no-build; \
    jupyter labextension install jupyterlab-jupytext --no-build; \
    jupyter labextension install jupyter-matplotlib --no-build; \
    jupyter labextension install @jupyterlab/toc --no-build;\
    jupyter labextension install jupyterlab-spreadsheet --no-build;\
    jupyter labextension install ipysheet --no-build;\
    jupyter labextension install @jupyterlab/dataregistry-extension --no-build; \
    jupyter labextension install @jupyterlab/shortcutui --no-build; \
    jupyter serverextension enable jupyterlab_sql --py --sys-prefix; \
    jupyter lab build; \
    npm cache clean --force; \
    rm -rf /home/$NB_USER/.cache/yarn; \
    rm -rf /home/$NB_USER/.node-gyp; \
    fix-permissions /etc/jupyter/; \
    rm -rf /tmp/*; \
)

# Import matplotlib the first time to build the font cache.
# A new RUN because installing R packages takes a long time
ENV XDG_CACHE_HOME=/home/$NB_USER/.cache/ \
    R_LIBS_USER=/home/jovyan/R/x86_64-redhat-linux-gnu-library/3.6

RUN mkdir -p /usr/share/doc/R/html/ && \
    chmod -R 777 /usr/share/doc/R  && \
    MPLBACKEND=Agg python3 -c "import matplotlib.pyplot" && \
    Rscript -e 'options(download.file.method = "libcurl")' && \
    Rscript -e "install.packages(c('httr','rmarkdown', 'IRkernel', 'factset.analyticsapi.engines'))" && \
    Rscript -e "IRkernel::installspec(user=FALSE)" && \
    mkdir -p /home/jovyan/.local/lib/python3.7/site-packages && \
    mkdir -p /home/jovyan/R/x86_64-redhat-linux-gnu-library/3.6 && \
    fix-permissions /home/$NB_USER

USER $NB_USER
RUN echo 'graphics_toolkit ("gnuplot")' >> ~/.octaverc

ENV CURL_CA_BUNDLE=/etc/pki/tls/certs/ca-certificates.crt \
    GIT_SSL_CAINFO=/usr/share/pki/ca-trust-source/ca-certificates.crt \
    LD_LIBRARY_PATH=/usr/lib/oracle/19.5/client64/lib:$LD_LIBRARY_PATH \
    PYTHONPATH=$PYTHONPATH:/home/jovyan/.local/lib/python3.7/site-packages \
    PATH=$PATH:/home/jovyan/.local/bin \
    EDITOR=nano

EXPOSE 8888
ENTRYPOINT ["/usr/local/bin/tini", "-g", "--"]
CMD ["start-notebook.sh"]
