FROM golang:1.17.1-alpine3.14 as builder
RUN apk add --no-cache git ca-certificates make tzdata
COPY . /app
RUN cd /app && \
    go get -d -v && \
    CGO_ENABLED=0 GOOS=linux go build -v -a -installsuffix cgo -o prometheus_bot

FROM alpine:3.13.6
COPY --from=builder /app/prometheus_bot /
RUN apk add --no-cache ca-certificates tzdata tini
RUN mkdir /etc/telegrambot/
USER nobody
EXPOSE 9087
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["/prometheus_bot","-c","/etc/telegrambot/prometheus-bot.yml","-d"]

