FROM docker.arvancloud.ir/library/ubuntu:24.04	

# Update package lists and install essential packages
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y apt-transport-https ca-certificates curl gnupg tzdata && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Set timezone
ENV TZ=Asia/Tehran
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

RUN apt-get update && \
    apt-get install -y apt-transport-https ca-certificates dirmngr && \
    apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754 && \
    echo "deb https://packages.clickhouse.com/deb stable main" | tee /etc/apt/sources.list.d/clickhouse.list && \
    apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y clickhouse-server clickhouse-client && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Set ownership to root: https://github.com/ClickHouse/ClickHouse/issues/6289
RUN chown -R root:root /var/lib/clickhouse
ENV CLICKHOUSE_CONFIG=/etc/clickhouse-server/config.xml