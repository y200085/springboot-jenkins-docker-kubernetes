FROM jenkins/jenkins:lts-jdk21

USER root

# Install Maven, Docker CLI, curl and certificates. The Docker daemon stays on Docker Desktop.
RUN apt-get update \
    && apt-get install -y --no-install-recommends maven docker.io curl ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Install kubectl from the official Kubernetes package repository.
RUN curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key \
      -o /tmp/kubernetes-release.key \
    && install -m 0755 -d /etc/apt/keyrings \
    && cat /tmp/kubernetes-release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg \
    && chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg \
    && echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' \
      > /etc/apt/sources.list.d/kubernetes.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends kubectl \
    && rm -rf /var/lib/apt/lists/* /tmp/kubernetes-release.key

USER jenkins
