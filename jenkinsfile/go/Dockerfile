FROM 10.199.10.237/bash/alpine-glibc:glibc-2.35
WORKDIR /opt/workdir/
COPY go-project /opt/workdir/
COPY conf/ /opt/workdir/conf/
ENV TZ=Asia/Shanghai
EXPOSE 8080
CMD [ "./go-project"]