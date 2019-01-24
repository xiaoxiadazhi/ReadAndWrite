## tencentHub

### image、容器、Docker Registry、Docker Hub关系

* image是由一堆只读层构成。每一层对应Dockerfile中的一条。
* 容器和image主要区别在于在镜像的一堆只读层上多了个可写层，任何修改都会保存在这个可写层，容器被删除的时候，可写层也一并删除，但底层镜像不会改变。
  * /var/lib/docker/aufs可以看层信息
  * https://docs.docker.com/v17.09/engine/userguide/storagedriver/imagesandcontainers/
  * https://segmentfault.com/a/1190000009309347
* Docker Registry就是存储image的地方，可以把image和Registry的关系与本地的源码和git服务的关系类比。
* Docker Registry和Docker Hub也可以类比Git和GitHub。前者是一个服务端程序，后者是网站。后者多了UI、鉴权等

### Docker Registry

* docker在本地如何管理image
  * https://segmentfault.com/a/1190000009730986
* 概述
  * https://linux.cn/thread-15605-1-1.html
* 官方文档
  * https://docs.docker.com/registry/
* v1 to v2
  * http://dockone.io/article/747
* registry存储结构
  * https://golangtc.com/t/56400487b09ecc72c3000021
* distribution代码中核心概念
  * manifest
    * 与v1到v2的改进有关，里面记录了所有层的关系，pull时先拉取manifest文件以后，可以并行拉取镜像层，提高效率
    * 官方文档https://docs.docker.com/registry/spec/manifest-v2-2/
  * digest
    * 基于内容进行寻址（Content-addressable)算法算出来的一串hash值
      * 内容不同得到的digest不同，内容相同得到的digest一定相同
    * docker push镜像时生成的
  * repository
  * blob
  * tag
  * reference
* distribution架构
  * API	-> 访问控制 ->handler -> storage

### tencentHub

.
├ ─ ─



