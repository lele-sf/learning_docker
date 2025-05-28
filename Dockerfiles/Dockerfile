FROM ubuntu:18.04
LABEL maintainer="email@gmail.com"
RUN apt-get update && apt-get install nginx -y
EXPOSE 80
COPY index.html /var/www/html/
WORKDIR /var/www/html
ENV APP_VERSION 1.0.0
ENTRYPOINT ["nginx"]
CMD ["nginx", "-g", "daemon off;"]
