### Tcr内网访问状态优化测试报告

#### 需求简述

Tcr1.0期，内网访问只有2个状态：

- `Running`（链路正常）
- `Unhealthy`（链路异常）

Tcr1.0期，新增一个状态，并对状态之间的变化做了限制：

- 新增状态：`Creating`（创建中）
- 创建内网访问时，状态显示`Creating`，之后会一直等到`Running`为止
- `Creating`到`Running`之间出现任何问题都会一直处于`Creating`（参考的tke集群创建），所以长期处于`Creating`属于异常情况
- `Creating`不会变成`Unhealthy`
- `Running`后由于某些原因导致running pod的个数不等于期望个数了，就会变成`Unhealthy`
- `Unhealthy`以后，如果running pod的个数等于期望个数了，又会变为`Running`



#### 测试例与测试结果

测试例：

1. 创建后先显示`Creating`
2. 一直处于`Creating`
3. `Creating` -> `Running` 
4. `Running` -> `Unhealthy`
5. `Unhealty` -> `Running`



测试结果：

1、创建后的显示`Creating`

- 包括部分异常时一直处于`Creating`（eni-controller有问题导致）

![tcr-creating](.\tcr-一直处于creating.png)



2、`Creating` -> `Running` 或`Unhealty` -> `Running`

![tcr-running](.\tcr-running.png)



3、`Running` -> `Unhealthy`

![tcr-unhealty](.\tcr-unhealty.png)