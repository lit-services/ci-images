# ci-images

Basis-Images für unsere CI. Bewusst ein eigenes Repository: ein Image, das den
Bau anderer Projekte ermöglicht, darf nicht in einem dieser Projekte liegen —
sonst hängt deren Bau an einem Artefakt, das sich mit ihnen mitversioniert.

Die Pakete sind **öffentlich**. Es steckt nichts Eigenes darin, und öffentlich
erspart dem Cluster ein `imagePullSecret` in jedem Namespace sowie die
Transfer-Grenzen des Free-Plans bei selbst-gehosteten Runnern.

## kaniko-glibc

```
ghcr.io/lit-services/kaniko-glibc:v1.23.2
ghcr.io/lit-services/kaniko-glibc:latest
```

Dieselben kaniko-Binaries wie im offiziellen Image, aber auf Debian statt
busybox.

**Wofür.** Unsere selbst-gehosteten Runner laufen im Kubernetes-Modus: jeder
Job bekommt einen eigenen Container, und der Runner reicht seine
Node-Binaries unter `/__e` hinein. Die sind gegen glibc gebaut. Nimmt man das
offizielle `gcr.io/kaniko-project/executor:*-debug` als Job-Container, startet
Node dort nicht:

```
env: can't execute '/__e/node24/bin/node': No such file or directory
```

Die Meldung führt in die Irre — die Datei ist vorhanden, es fehlt der
ELF-Interpreter, weil das Image musl-basiert ist. Mit Debian als Basis
verschwindet das Problem.

**Verwendung** in einem Workflow auf einem Runner im Kubernetes-Modus:

```yaml
jobs:
  build:
    runs-on: k8s-ci-docker
    container:
      image: ghcr.io/lit-services/kaniko-glibc:v1.23.2
    steps:
      - uses: actions/checkout@v4
      - run: |
          /kaniko/executor \
            --context "$GITHUB_WORKSPACE" \
            --dockerfile "$GITHUB_WORKSPACE/Dockerfile" \
            --no-push --cache=false
```

**Wartung.** Der Workflow baut bei Änderungen an `kaniko-glibc/` und einmal
monatlich neu, damit die Debian-Basis ihre Sicherheitsaktualisierungen
bekommt. Die kaniko-Version steht als `KANIKO_VERSION` an einer Stelle im
Workflow und wird ans Dockerfile durchgereicht.

Der Build prüft am Ende, dass glibc vorhanden und der Executor lauffähig ist —
ein erfolgreicher Build allein wäre kein Nachweis für die eine Eigenschaft,
derentwegen es das Image gibt.
