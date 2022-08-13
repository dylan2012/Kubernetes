FROM nginx:stable-alpine
WORKDIR /usr/share/nginx/html/
COPY dist/ /usr/share/nginx/html/
ENV TZ=Asia/Shanghai
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]