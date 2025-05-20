FROM eclipse-temurin:21-jre

RUN apt-get update && apt-get install unzip -y

WORKDIR /opt
COPY trafficparrot-license-usage-*.zip trafficparrot-license-usage.zip
RUN unzip trafficparrot-license-usage.zip && rm trafficparrot-license-usage.zip && mv trafficparrot-license-usage-* trafficparrot-license-usage
COPY trafficparrot.usage.license /opt/trafficparrot-license-usage
COPY licenseusage.properties /opt/trafficparrot-license-usage

RUN chgrp -R 0 /opt/trafficparrot-license-usage && chmod -R g=u /opt/trafficparrot-license-usage

WORKDIR /opt/trafficparrot-license-usage
VOLUME ["/opt/trafficparrot-license-usage/data"]
CMD ["./start.sh"]
