FROM ubuntu:24.04 as builder

ARG BASE_URL
ARG HUGO_VERSION

ADD https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_Linux-64bit.tar.gz /tmp
RUN tar -xf /tmp/hugo_${HUGO_VERSION}_Linux-64bit.tar.gz -C /usr/local/bin

COPY . /source
RUN hugo --gc --minify --source /source/ --destination /public/ -b ${BASE_URL}

FROM nginx:1.25.4-alpine3.18-perl

RUN rm /usr/share/nginx/html/*
COPY --from=builder /public/ /usr/share/nginx/html/
