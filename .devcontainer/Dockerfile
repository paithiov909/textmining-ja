ARG VARIANT="4"
FROM ghcr.io/rocker-org/devcontainer/tidyverse:${VARIANT}

RUN python3 -m pip install black flake8 ipadic && \
    python3 -c "import ipadic; print('dicdir=' + ipadic.DICDIR);" > /usr/local/etc/mecabrc
