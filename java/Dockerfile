FROM maven:alpine as build
ENV HOME=/usr/app
RUN mkdir -p $HOME
WORKDIR $HOME
#m2依赖文件拷贝 add自动解压
#ADD m2.tar /
#添加源码
ADD . $HOME
RUN mvn package -nsu -P dev -Dmaven.test.skip=true

FROM openjdk:8-jdk-alpine
MAINTAINER dylan
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
COPY --from=build /usr/app/target/api-0.0.1-SNAPSHOT.jar /app/app.jar
#ADD app.jar /app/app.jar
EXPOSE 8080
ENTRYPOINT java -jar /app/app.jar
