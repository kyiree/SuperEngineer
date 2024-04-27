# 1 主从环境搭建

采用 docker-compose 的方式搭建一主一从的 redis 环境进行学习：

```
version: '3'

services:
  redis-master:
    image: redis:latest
    container_name: redis-master
    ports:
      - "6379:6379"
    networks:
      - redis-net
    command: redis-server --appendonly yes

  redis-slave:
    image: redis:latest
    container_name: redis-slave
    ports:
      - "6380:6379"
    depends_on:
      - redis-master
    networks:
      - redis-net
    command: redis-server --slaveof redis-master 6379 --appendonly yes

networks:
  redis-net:
    driver: bridge
```

docker-compose -f xxx.yml up -d

主机可读可写，从机只能读不能写，主机写完数据后，会将数据同步给从机

# 2 主从复制底层原理

主节点写数据的时候，同时将需要同步的命令写到主节点的复制积压缓冲区和各个从节点的输出缓冲区，事件循环的时候，会判断缓冲区是否有数据可写，如果有就将这里的数据发给从节点
