
FROM jenkinsci/jenkins:2.30

USER root

RUN apt-get update -qq && apt-get install -qqy \
    apt-transport-https \
    ca-certificates \
    curl \
    wget \
    lxc \
    iptables \
    dos2unix
    

    
RUN /usr/local/bin/install-plugins.sh workflow-aggregator












#Ref: 
#Jenkins docker images: https://hub.docker.com/r/jenkinsci/jenkins/tags/
#Best Practice Docke file:  https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/
