bug: springboot 配置 logback-spring.xml 时 不生效 无法将日志写入文件夹

solve: 需在启动脚本中指定spring.profiles.active=xxx, 在application.yml中指定无效.(太神奇)