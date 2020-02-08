FROM fedora:latest

LABEL maintainer="dmpe <cincenko@outlook.com>"
ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"

USER root

RUN (set -ex ;\ 
    dnf update -y; \
    dnf upgrade -y; \
    dnf groupinstall "Development Tools" "Development Libraries" -y; \
)
RUN (set -e; \
    dnf install -y R R-devel czmq-devel python*-networkx git tree mlocate jq bzip2 liberation-fonts dejavu-fonts-common icu4j-localespi libicu wget \
        tig pandoc python3-devel tzdata unzip nano tcl tk gfortran gcc-gfortran;\
    curl -sL https://rpm.nodesource.com/setup_13.x | bash - ; \
    python3 -m pip install numpy scipy matplotlib ipython pandas sympy nose \
        beautifulsoup4 bokeh cloudpickle cython dask dill ipywidgets matplotlib numba \
        numexpr patsy protobuf scikit-image scikit-learn seaborn sqlalchemy statsmodels vincent xlrd \
        openpyxl xlsxwriter notebook jupyterhub jupyterlab;\
    curl -sL https://dl.yarnpkg.com/rpm/yarn.repo | tee /etc/yum.repos.d/yarn.repo ;\
    yum install -y yarn nodejs ;\
)

ENV LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8 \
    NB_USER=$NB_USER \
    NB_UID=$NB_UID \
    NB_GID=$NB_GID \
    HOME=/home/$NB_USER

#RUN localedef -v -c -i en_US -f UTF-8 en_US.UTF-8

# Copy a script that we will use to correct permissions after running certain commands
COPY fix-permissions /usr/local/bin/fix-permissions
RUN chmod a+rx /usr/local/bin/fix-permissions

# Enable prompt color in the skeleton .bashrc before creating the default NB_USER
RUN sed -i 's/^#force_color_prompt=yes/force_color_prompt=yes/' /etc/skel/.bashrc

# Create NB_USER wtih name jovyan user with UID=1000 and in the 'users' group
# and make sure these dirs are writable by the `users` group.
RUN echo "auth requisite pam_deny.so" >> /etc/pam.d/su && \
    sed -i.bak -e 's/^%admin/#%admin/' /etc/sudoers && \
    sed -i.bak -e 's/^%sudo/#%sudo/' /etc/sudoers && \
    useradd -m -s /bin/bash -N -u $NB_UID $NB_USER && \
    chmod g+w /etc/passwd && \
    fix-permissions $HOME

ENV TINI_VERSION v0.18.0
ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /usr/local/bin/tini
RUN chmod +x /usr/local/bin/tini

USER $NB_UID
RUN mkdir /home/$NB_USER/work && \
    mkdir /home/$NB_USER/.jupyter && \
    echo "cacert=/etc/ssl/certs/ca-certificates.crt" > /home/$NB_USER/.curlrc
WORKDIR $HOME
RUN chown -R $NB_USER:users /home/$NB_USER/.jupyter

# Setup work directory for backward-compatibility
RUN fix-permissions /home/$NB_USER

RUN npm cache clean --force && \
    jupyter notebook --generate-config && \
    fix-permissions /home/$NB_USER

EXPOSE 8888

# Configure container startup
ENTRYPOINT ["/usr/local/bin/tini", "-g", "--"]
CMD ["start-notebook.sh"]

# Copy local files as late as possible to avoid cache busting
COPY start.sh /usr/local/bin/
COPY start-notebook.sh /usr/local/bin/
COPY start-singleuser.sh /usr/local/bin/
COPY jupyter_notebook_config.py /etc/jupyter/

# Fix permissions on /etc/jupyter as root
USER root
RUN fix-permissions /etc/jupyter/

# Switch back to jovyan to avoid accidental container runs as root
USER $NB_UID

# Import matplotlib the first time to build the font cache.
ENV XDG_CACHE_HOME /home/$NB_USER/.cache/
RUN MPLBACKEND=Agg python3 -c "import matplotlib.pyplot" && \
    fix-permissions /home/$NB_USER

RUN cd /tmp && \
    git clone https://github.com/PAIR-code/facets.git && \
    cd facets && \
    jupyter nbextension install facets-dist/ --sys-prefix && \
    cd && \
    rm -rf /tmp/facets && \
    fix-permissions /home/$NB_USER

# Activate ipywidgets extension in the environment that runs the notebook server
RUN (set -e; \ 
    jupyter nbextension enable --py widgetsnbextension --sys-prefix; \
    # Also activate ipywidgets extension for JupyterLab
    # Check this URL for most recent compatibilities
    # https://github.com/jupyter-widgets/ipywidgets/tree/master/packages/jupyterlab-manager
    jupyter labextension install @jupyter-widgets/jupyterlab-manager@^1.0.1 --no-build; \
    jupyter labextension install jupyterlab_bokeh --no-build; \
    jupyter lab build; \
    npm cache clean --force; \
    rm -rf /home/$NB_USER/.cache/yarn;\
    rm -rf /home/$NB_USER/.node-gyp; \
    fix-permissions /home/$NB_USER; \
)

RUN Rscript -e 'options(repos="https://cloud.r-project.org")'
RUN Rscript -e "install.packages(c('caret', 'crayon', 'devtools', 'forecast', 'hexbin', 'htmltools', 'htmlwidgets', 'irkernel', 'plyr', 'randomforest', 'rcurl', 'reshape2', 'rmarkdown', 'rsqlite', 'shiny', 'tidyverse','rpy2'))"
RUN fix-permissions /home/$NB_USER