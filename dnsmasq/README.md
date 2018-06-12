# Docker dnsmasq container

## Dockerfile

### 
    cat > Dockerfile  <<EOF
    FROM alpine:edge
    RUN apk --no-cache add dnsmasq
    EXPOSE 53 53/udp
    ENTRYPOINT ["dnsmasq", "-k"]
    EOF

    
## run

### 
    docker run --name dnsmasq -d --privileged --net=host --cap-add=NET_ADMIN -v /etc/:/etc/ -v /var/:/var/ --restart always dnsmasq --log-facility=/var/log/dnsmasq.log