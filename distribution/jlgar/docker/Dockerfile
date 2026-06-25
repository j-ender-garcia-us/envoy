FROM debian:12-slim

ARG ENVOY_BINARY=envoy-static.opt

RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates \
    && rm -rf /var/lib/apt/lists/*

COPY ${ENVOY_BINARY} /usr/local/bin/envoy
COPY envoy.yaml /etc/envoy/envoy.yaml

RUN chmod 0755 /usr/local/bin/envoy

EXPOSE 80 443 9901

ENTRYPOINT ["/usr/local/bin/envoy"]
CMD ["-c", "/etc/envoy/envoy.yaml", "--concurrency", "2", "-l", "info"]
