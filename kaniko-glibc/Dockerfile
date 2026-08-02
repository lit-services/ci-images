# kaniko auf glibc.
#
# Warum es das braucht: unsere CI läuft auf selbst-gehosteten Runnern im
# Kubernetes-Modus. Dort führt jeder Job in einem eigenen Container aus, und
# der Runner reicht seine mitgelieferten Node-Binaries hinein (/__e). Die sind
# gegen glibc gebaut.
#
# Das offizielle kaniko-Image `gcr.io/kaniko-project/executor:*-debug` ist
# busybox-basiert (musl). Nimmt man es als Job-Container, startet das
# hineingereichte Node nicht:
#
#   env: can't execute '/__e/node24/bin/node': No such file or directory
#
# „No such file" ist dabei irreführend — die Datei ist da, es fehlt der
# ELF-Interpreter. Deshalb hier dieselben kaniko-Binaries auf einer
# Debian-Basis.
ARG KANIKO_VERSION=v1.23.2

FROM gcr.io/kaniko-project/executor:${KANIKO_VERSION}-debug AS kaniko

FROM debian:bookworm-slim

# ca-certificates: kaniko lädt Basis-Images über HTTPS.
# git: für Kontexte, die aus einem Repository kommen.
RUN apt-get update \
    && apt-get install -y --no-install-recommends ca-certificates git \
    && rm -rf /var/lib/apt/lists/*

# Der Executor ist ein statisch gelinktes Go-Binary und läuft deshalb auch
# außerhalb seines Ursprungs-Images. Mitkopiert werden die Verzeichnisse, die
# kaniko zur Laufzeit erwartet.
COPY --from=kaniko /kaniko /kaniko

ENV PATH=/kaniko:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
    DOCKER_CONFIG=/kaniko/.docker \
    SSL_CERT_DIR=/kaniko/ssl/certs

# Bewusst kein ENTRYPOINT auf den Executor: als Job-Container einer
# GitHub-Action muss das Image eine Shell offenhalten, damit der Runner seine
# Schritte hineinschicken kann. Der Aufruf erfolgt im Workflow-Schritt.
CMD ["/bin/sh"]
