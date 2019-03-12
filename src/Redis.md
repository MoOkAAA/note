#Redis
Docker 启动 redis daemonize 不能开启 (无法已守护进程形式启动)

why

Docker启动时 当第一个进程结束时 容器就会退出(exited)
而 redis 开启 daemonize时 会先正常启动redis 进程 再开始一个daemon进程 随后结束先开启的进程 此时容器就会结束

docker 启动 redis时 bind 需改为0.0.0.0

redis 缓存时

注意 @Cacheable @CachePut 的 key要一致 修改时才能修改缓存 否则会新增缓存

注意 @Cacheable 中要加入 null时 不缓存 否则null也会缓存

分布式加锁

setIfAbsent key进redis 如果redis中无则 put进去 也就相印的获取到锁 解锁时 del该key