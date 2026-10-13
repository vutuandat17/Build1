# =============================================
# Image Jenkins cho cách cấu hình qua PORTAL
# Chỉ là Jenkins LTS gốc + Docker CLI (để pipeline build image / deploy).
# KHÔNG cài sẵn plugin, tool hay cấu hình nào: tất cả làm trên giao diện web
# (Setup Wizard, Plugins, Tools, Credentials, System) theo README.md.
# =============================================
ARG JENKINS_VERSION=2.568.3
FROM jenkins/jenkins:${JENKINS_VERSION}-lts-jdk21

USER root

# Docker CLI + compose plugin – dùng Docker của máy host qua /var/run/docker.sock
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl gnupg \
 && install -m 0755 -d /etc/apt/keyrings \
 && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
      > /etc/apt/sources.list.d/docker.list \
 && apt-get update \
 && apt-get install -y --no-install-recommends docker-ce-cli docker-buildx-plugin docker-compose-plugin \
 && rm -rf /var/lib/apt/lists/*

USER jenkins
