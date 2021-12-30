FROM golang:1.17.1-bullseye

WORKDIR /home

COPY ./pkg /home

RUN cd /home && \
    go build -o library

CMD ["/home/library"]