redis 是mysql的盾牌，缓存常用数据，防止大量连接贯穿到 db，导致宕机

redis 与 memcache的区别

1 redis 不仅支持 string ，还支持 list set、zset 、hash

2 redis可持久化也可以基于内存

3  