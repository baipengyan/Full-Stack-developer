# docker国内镜像源



**Docker中国区官方镜像:**
https://registry.docker-cn.com
**网易:**
http://hub-mirror.c.163.com
**ustc:**
https://docker.mirrors.ustc.edu.cn
**中国科技大学:**
https://docker.mirrors.ustc.edu.cn
**阿里云:**
https://cr.console.aliyun.com/

**配置docker**

> 为加快拉取镜像速度，建议设置docker国内镜像源

```
# 创建或修改 /etc/docker/daemon.json 文件，修改为如下形式
{
    "registry-mirrors" : [
    "https://registry.docker-cn.com",
    "https://docker.mirrors.ustc.edu.cn",
    "http://hub-mirror.c.163.com",
    "https://cr.console.aliyun.com/"
  ]
}
# 重启docker服务使配置生效
$ systemctl restart docker.service
```