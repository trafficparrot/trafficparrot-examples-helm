FROM eclipse-temurin:21-jre

RUN apt-get update && apt-get install unzip -y

WORKDIR /opt
COPY trafficparrot-*.zip trafficparrot.zip
RUN unzip trafficparrot.zip && rm trafficparrot.zip && mv trafficparrot-* trafficparrot
COPY trafficparrot.license /opt/trafficparrot
COPY trafficparrot.properties /opt/trafficparrot

RUN chgrp -R 0 /opt/trafficparrot && chmod -R g=u /opt/trafficparrot

WORKDIR /opt/trafficparrot
CMD ["./start-foreground.sh"]