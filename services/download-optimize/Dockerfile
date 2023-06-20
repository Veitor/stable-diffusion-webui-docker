# FROM registry-vpc.cn-hangzhou.aliyuncs.com/hangjia/aria2:1.36
FROM bash:alpine3.15
RUN apk add parallel aria2
COPY . /docker
RUN chmod +x /docker/download.sh
ENTRYPOINT ["/docker/download.sh"]
