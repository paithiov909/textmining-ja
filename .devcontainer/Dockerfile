# syntax=docker/dockerfile:1

ARG VARIANT="4"
FROM ghcr.io/rocker-org/devcontainer/r-ver:${VARIANT}

ENV LANG="ja_JP.UTF-8"
ENV LC_ALL="ja_JP.UTF-8"

RUN apt-get -y update && export DEBIAN_FRONTEND=noninteractive && apt-get -y --no-install-recommends install \
    fonts-noto-cjk \
    fonts-ipafont-gothic \
    libmecab-dev \
    mecab \
    mecab-ipadic-utf8 && \
    apt-get autoremove -y && apt-get clean && rm -rf /var/cache/apt/archives/* /var/lib/apt/lists/*

RUN sed -i '$d' /etc/locale.gen \
    && echo "ja_JP.UTF-8 UTF-8" >> /etc/locale.gen \
    && locale-gen ja_JP.UTF-8 \
    && /usr/sbin/update-locale LANG=ja_JP.UTF-8 LANGUAGE="ja_JP:ja" \
    && /bin/bash -c "source /etc/default/locale" \
    && ln -sf /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
