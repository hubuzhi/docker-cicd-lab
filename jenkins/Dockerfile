FROM jenkins/jenkins:lts
USER root
RUN curl -sSL https://mirrors.aliyun.com/docker-ce/linux/static/stable/x86_64/docker-27.3.1.tgz -o /tmp/d.tgz \
 && tar -xzf /tmp/d.tgz -C /tmp \
 && cp /tmp/docker/docker /usr/local/bin/docker \
 && chmod +x /usr/local/bin/docker \
 && rm -rf /tmp/d.tgz /tmp/docker \
 && docker --version
USER jenkins
