FROM redis:8.10.3

COPY redis.conf /usr/local/etc/redis/redis.conf
COPY redis-fast.conf /usr/local/etc/redis/redis-fast.conf

CMD ["redis-server", "/usr/local/etc/redis/redis.conf"]
