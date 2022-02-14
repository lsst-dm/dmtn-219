FROM lsstsqre/centos:w_latest
ENV PYTHONUNBUFFERED True
RUN source /opt/lsst/software/stack/loadLSST.bash \
    && conda install -y mamba \
    && mamba install -y \
        flask \
        gunicorn \
        google-cloud-pubsub \
        google-cloud-sdk \
        google-cloud-storage
