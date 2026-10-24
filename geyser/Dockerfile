# Geyser Standalone — translates Bedrock clients into Java clients and hands
# them to mc-router. See the README "Bedrock support" section.
#
# eclipse-temurin:25-jre has no curl/wget, so ADD fetches the jar at build time.
# The build is pinned for reproducibility; bump the URL to upgrade.
FROM eclipse-temurin:25-jre
ADD https://download.geysermc.org/v2/projects/geyser/versions/2.12.0/builds/1258/downloads/standalone /opt/geyser/Geyser-Standalone.jar
COPY config.yml /opt/geyser/config.yml
WORKDIR /geyser
# Geyser rewrites its config on boot, so copy the repo version in fresh every start
CMD ["sh", "-c", "cp /opt/geyser/config.yml /geyser/config.yml && exec java -Xms256M -Xmx1G -jar /opt/geyser/Geyser-Standalone.jar --nogui"]
