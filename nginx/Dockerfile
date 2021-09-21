FROM debian:buster-slim

ENV  SONIX_SHA256    e5a01ee9735febc2d6ef0f99806a47bd359b0c8eb7731867868d6b044ec0ca9e

RUN  apt-get update && apt-get -y upgrade && apt-get -y install wget nginx-common libgeoip-dev \
     && cd && wget https://github.com/techwise-technologies/sonix/releases/download/v1.19.3/sonix-1.19.3.amd64-linux-gnu.deb \
     && echo "$SONIX_SHA256 sonix-1.19.3.amd64-linux-gnu.deb" | sha256sum -c - && \
     dpkg -i sonix-1.19.3.amd64-linux-gnu.deb \
     && rm -rf sonix-1.19.3.amd64-linux-gnu.deb

COPY ./nginx/nginx.conf /etc/nginx/nginx.conf

EXPOSE 80 443

STOPSIGNAL SIGQUIT

CMD ["sonix", "-g", "daemon off;"]
