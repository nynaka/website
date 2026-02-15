# Base Image
FROM rockylinux:9

# systemd のインストール
RUN dnf update -y \
    && dnf install -y systemd

# procps-ng: ps コマンド
RUN dnf install -y \
    procps-ng

# 基本ツールのインストール
RUN dnf update -y
RUN dnf install -y \
    sudo glibc-langpack-ja \
    git vim jq

# ロケールの設定
ENV LANG=ja_JP.UTF-8 \
    LC_ALL=ja_JP.UTF-8

COPY vimrc /root/.vimrc

CMD ["/sbin/init"] 
