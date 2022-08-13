FROM openjdk:11-jdk-slim
MAINTAINER dylan
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
ENV HOME=/app
RUN mkdir -p $HOME
WORKDIR $HOME
ADD app.jar $HOME/app.jar
EXPOSE 8080
ENTRYPOINT java -jar $HOME/app.jar