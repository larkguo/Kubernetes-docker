
# Docker keepalived container

## Dockerfile

### cat > Dockerfile  <<EOF
FROM alpine:edge
RUN apk --no-cache add keepalived
ENTRYPOINT ["/usr/sbin/keepalived"]
CMD ["--dont-fork", "--log-console"]
EOF

    
## run

### docker run --name keepalived -d --restart always --cap-add=NET_ADMIN --privileged --network=host -v /etc/keepalived/:/etc/keepalived/  keepalived