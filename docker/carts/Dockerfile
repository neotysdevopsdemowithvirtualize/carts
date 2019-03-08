FROM weaveworksdemos/msd-java:jre-latest

WORKDIR /usr/src/app
COPY *.jar ./app.jar

RUN	chown -R ${SERVICE_USER}:${SERVICE_GROUP} ./app.jar

USER ${SERVICE_USER}


ARG BUILD_VERSION
ARG COMMIT

LABEL org.label-schema.vendor="Weaveworks" \
  org.label-schema.version="${BUILD_VERSION}" \
  org.label-schema.name="Socks Shop: Cart" \
  org.label-schema.description="REST API for Cart service" \
  org.label-schema.url="https://github.com/neotysdevopsdemo-demo/carts" \
  org.label-schema.vcs-url="github.com:neotysdevopsdemo/carts.git" \
  org.label-schema.vcs-ref="${COMMIT}"

ENTRYPOINT ["/usr/local/bin/java.sh","-jar","./app.jar", "--port=80"]
