# Preview-only build/run image for the Kill Bill server.
#
# Kill Bill's own dev workflow (bin/start-server) runs the profiles/killbill WAR with
# `mvn jetty:run` (embedded Jetty via the jetty-maven-plugin) -- there is no fat/executable
# jar produced for this module. This Dockerfile reproduces that same flow unchanged:
#   1. build this module and its reactor dependencies (mirrors .github/workflows/ci.yml's
#      build step and bin/start-server's check.skip-* flags), and
#   2. start it with `mvn jetty:run`, the same command bin/start-server uses, listening on
#      the platform-assigned $PORT via the jetty-maven-plugin's `jetty.port` system property.
#
# DB connection and RBAC/tenant config are supplied entirely via env vars at runtime (Kill
# Bill's own KB_-prefixed environment variable override, enabled below with
# -Dorg.killbill.server.lookupEnvironmentVariables=true) -- no application config file is
# modified.
FROM maven:3.9-eclipse-temurin-21

WORKDIR /app
COPY . .

RUN mvn -B --show-version clean install -DskipTests=true \
        -Dcheck.skip-dependency=true \
        -Dcheck.skip-dependency-scope=true \
        -Dcheck.skip-dependency-versions=true \
        -Dcheck.skip-duplicate-finder=true \
        -Dcheck.skip-enforcer=true \
        -Dcheck.skip-rat=true \
        -Dcheck.skip-spotbugs=true \
        -pl profiles/killbill -am

WORKDIR /app/profiles/killbill

# DB_HOST is the private preview-db service's internal hostname (see plan envVars). It only
# registers in internal DNS once that service is live, so wait for it to actually accept
# connections before starting Kill Bill (which opens its datasource pool at boot).
CMD ["bash", "-c", "if [ -n \"$DB_HOST\" ]; then i=0; while [ $i -lt 60 ]; do (echo > /dev/tcp/$DB_HOST/5432) >/dev/null 2>&1 && break; i=$((i+1)); sleep 5; done; fi; mvn -B -Dcom.sun.xml.bind.v2.bytecode.ClassTailor.noOptimize=true -Dorg.killbill.server.properties=file://$(pwd)/src/main/resources/killbill-server.properties -Dlogback.configurationFile=file://$(pwd)/src/main/resources/logback.xml -Dcheck.skip-dependency=true -Dcheck.skip-dependency-scope=true -Dcheck.skip-dependency-versions=true -Dcheck.skip-duplicate-finder=true -Dcheck.skip-enforcer=true -Dcheck.skip-rat=true -Dcheck.skip-spotbugs=true -Dorg.killbill.server.lookupEnvironmentVariables=true -Djetty.port=${PORT:-8080} jetty:run"]
