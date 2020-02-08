FROM ubuntu:18.04

# Install JDK, misc packages.
RUN apt-get update && apt-get install -y \
    default-jdk \
    apt-transport-https \
    curl

RUN curl -L https://github.com/lihaoyi/Ammonite/releases/download/2.0.4/2.13-2.0.4 > /usr/local/bin/amm && chmod +x /usr/local/bin/amm && mkdir /root/.ammonite

COPY ./predef.sc /root/.ammonite/predef.sc

RUN export TERM=vt100

CMD amm
