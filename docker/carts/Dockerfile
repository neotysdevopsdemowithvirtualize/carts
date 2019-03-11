FROM java:openjdk-8-alpine

WORKDIR /usr/src/app
COPY *.jar ./app.jar

ARG BUILD_VERSION
ARG COMMIT

LABEL org.label-schema.vendor="Neotys" \
  org.label-schema.version="${BUILD_VERSION}" \
  org.label-schema.name="Socks Shop: Cart" \
  org.label-schema.description="REST API for Cart service" \
  org.label-schema.url="https://github.com/neotysdevopsdemo/carts" \
  org.label-schema.vcs-url="github.com:neotysdevopsdemo/carts.git" \
  org.label-schema.vcs-ref="${COMMIT}"

ENTRYPOINT ["java","-Djava.security.egd=file:/dev/urandom","-jar","./app.jar", "--port=80"]





