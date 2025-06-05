FROM golang:1.18 AS buildando
WORKDIR /app
COPY . ./
RUN go mod init hello
RUN go build -o /app/hello

FROM alpine:3.15.9
# vai dar erro:
SHELL ["/bin/bash"]
COPY --from=buildando /app/hello /app/hello
ENV APP="hello_world"
ARG USER="some_user"
ENV USER=$USER
RUN echo "User: $USER"
VOLUME /app/dados
CMD ["/app/hello"]

# docker build -t go-teste:4.0 .
# docker build -t go-teste:4.0 --build-arg USER=other_user .
# docker run -ti go-teste:4.0
# docker ps -a
# docker container inspect <id-container>