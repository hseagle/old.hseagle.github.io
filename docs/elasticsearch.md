# elasticsearch

## Elasticsearch编译与安装

```gradle
repositories {
	maven { url 'https://repo.gradle.org/gradle/libs-releases-local' }
	maven { url "http://oss.sonatype.org/content/repositories/snapshots" }
	maven { url 'http://maven.aliyun.com/mvn/repository/' }
	jcenter { url "http://jcenter.bintray.com/"}
	maven { url "https://jitpack.io" }
}
```

```bash
gradle idea
```

导入到intellij idea

设置gradle的路径，如果是在macbook中路径设置为

- 什么是一个好的olap方案
    1. dynamic data, 支持增删改
    2. 实时性要强
    3. 支持水平扩展
    4. 强一致性
    5. schemaless
    6. 丰富的内嵌函数 built-in functions
    7. 支持SQL
    8. 速度快，可以接受的时延
    9. 好的社区支持
    10. 广泛的使用案例
    11. live aggregation
    12. friendly document
    13. json
    14. support index
		15. binary data storage，比如图像和音频的数据存储机制

- 收益
    1. 发现更多的事实，
        - 减少损失
        - 开展新业务
        - 制定新规则
    2. 节省时间
    3. 使用简单
