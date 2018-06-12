FROM alpine:edge
RUN apk --no-cache add keepalived
ENTRYPOINT ["/usr/sbin/keepalived"]
CMD ["--dont-fork", "--log-console"]
