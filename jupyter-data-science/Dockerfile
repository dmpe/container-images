FROM fedora:latest

LABEL maintainer="dmpe <cincenko@outlook.com>"
ARG NB_USER="jovyan"
ARG NB_UID="1000"
ARG NB_GID="100"

USER root

RUN (set -ex ;\ 
    dnf update -y; \
    dnf upgrade -y; \
    dnf groupinstall -y "Development Tools" "Development Libraries"; \
)
RUN (set -e; \
    dnf install -y R R-devel czmq-devel boost-devel QuantLib bash-completion python*-networkx tree mlocate jq bzip2 \
                   liberation-fonts dejavu-fonts-common icu4j-localespi wget octave octave-devel gnuplot \
                   graphviz-devel cairo-devel libjpeg-devel giflib-devel librsvg2-devel \
                   tig pandoc nano tcl tk gfortran gcc-gfortran compat-openssl10 unixODBC-devel;\
    curl -sL https://rpm.nodesource.com/setup_13.x | bash - ; \
    python3 -m pip install numpy scipy ipython notebook jupyterhub jupyterlab pandas bqplot octave_kernel bash_kernel \
        beautifulsoup4 bokeh cloudpickle cython dask jupytext jupyterlab_sql psutil ipywidgets matplotlib \
        numexpr protobuf ipysheet fds.analyticsapi.engines nbresuse nbdime scikit-learn seaborn sqlalchemy statsmodels vincent xlrd \
        openpyxl xlsxwriter jupyter_bokeh jupyterlab-git jupyterlab-commenting-service ipympl;\
    curl -sL https://dl.yarnpkg.com/rpm/yarn.repo | tee /etc/yum.repos.d/yarn.repo ;\
    dnf install -y yarn nodejs ;\
    python3 -m bash_kernel.install; \
)

RUN cd /tmp && curl -k -LOJ https://bloomberg.bintray.com/BLPAPI-Stable-Generic/blpapi_cpp_3.12.3.1-linux.tar.gz && \ 
     tar zxvf blpapi_cpp_3.12.3.1-linux.tar.gz && ls -al

RUN export BLPAPI_ROOT=/tmp/blpapi_cpp_3.12.3.1

ENV LC_ALL=en_US.UTF-8 \
    LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8 \
    NB_USER=$NB_USER \
    NB_UID=$NB_UID \
    NB_GID=$NB_GID \
    BLPAPI_ROOT=/tmp/blpapi_cpp_3.12.3.1 \
    HOME=/home/$NB_USER \
    TINI_VERSION=v0.18.0
    
RUN echo "LC_ALL=en_US.UTF-8" >> /etc/environment && \
    echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen && \
    echo "LANG=en_US.UTF-8" > /etc/locale.conf && \
    export LANG=en_US.UTF-8 && \
    export LC_ALL=C && \
    pip install --index-url=https://bloomberg.bintray.com/pip/simple blpapi && rm -rf /tmp/blpapi_cpp_3.12.3.1-linux.tar.gz && rm -rf /tmp/blpapi_cpp_3.12.3.1

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

ADD https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini /usr/local/bin/tini
RUN chmod +x /usr/local/bin/tini

USER $NB_UID
RUN export LC_ALL=C && mkdir /home/$NB_USER/work && \
    mkdir /home/$NB_USER/.jupyter && \
    echo "cacert=/etc/ssl/certs/ca-certificates.crt" > /home/$NB_USER/.curlrc
WORKDIR $HOME
RUN chown -R $NB_USER:users /home/$NB_USER/.jupyter && \
    npm cache clean --force && \
    jupyter notebook --generate-config

# Copy local files as late as possible to avoid cache busting
COPY start.sh /usr/local/bin/
COPY start-notebook.sh /usr/local/bin/
COPY start-singleuser.sh /usr/local/bin/
COPY jupyter_notebook_config.py /etc/jupyter/

# Fix permissions on /etc/jupyter as root
USER root
RUN fix-permissions /etc/jupyter/
RUN export LC_ALL=C && cd /tmp && \
    git clone https://github.com/PAIR-code/facets.git && \
    cd facets && \
    jupyter nbextension install facets-dist/ --sys-prefix && \
    cd && rm -rf /tmp/facets && \
    fix-permissions /home/$NB_USER

# Activate ipywidgets extension in the environment that runs the notebook server
RUN (set -e; \ 
    jupyter nbextension enable --py widgetsnbextension --sys-prefix; \
    # Also activate ipywidgets extension for JupyterLab
    # Check this URL for most recent compatibilities
    # https://github.com/jupyter-widgets/ipywidgets/tree/master/packages/jupyterlab-manager
    jupyter labextension install @jupyter-widgets/jupyterlab-manager@^1.0.1 --no-build; \
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
    jupyter labextension install @jupyterlab/commenting-extension --no-build; \
    jupyter serverextension enable jupyterlab_sql --py --sys-prefix; \
    jupyter lab build; \
    npm cache clean --force; \
    rm -rf /home/$NB_USER/.cache/yarn; \
    rm -rf /home/$NB_USER/.node-gyp; \
    fix-permissions /home/$NB_USER; \
)

RUN mkdir -p /usr/share/doc/R/html/ && chmod -R 777 /usr/share/doc/R
# Import matplotlib the first time to build the font cache.
ENV XDG_CACHE_HOME /home/$NB_USER/.cache/
RUN MPLBACKEND=Agg python3 -c "import matplotlib.pyplot" && \
    fix-permissions /home/$NB_USER

RUN Rscript -e "install.packages(c('caret', 'RQuantLib', 'devtools', 'htmltools', 'htmlwidgets', 'rmarkdown', 'tidyverse', 'IRkernel', 'factset.analyticsapi.engines'), repos='https://cloud.r-project.org')"

# Switch back to jovyan to avoid accidental container runs as root
USER $NB_UID
RUN export LC_ALL=C && echo 'graphics_toolkit ("gnuplot")' >> ~/.octaverc

# R Kernel must be installed locally, not with root
RUN Rscript -e "IRkernel::installspec()"

EXPOSE 8888
ENTRYPOINT ["/usr/local/bin/tini", "-g", "--"]
CMD ["start-notebook.sh"]
