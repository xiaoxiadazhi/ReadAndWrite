### 原逻辑

* 1、自己实现线程池，管理线程池是否繁忙
  * taskList，runningFile，ProcessTask，workerThread，threadPool
* 2、线程池大小为50，scandir以后，判断线程池是否繁忙（runningFile>50为busy），不繁忙往taskList中add任务，runnigFile中添加文件，处理完一个文件即从runningFile里面删除，添加操作都是sychronized的

### 分析过程

* 首先打印日志看哪些代码耗时
  * 得知文件150w左右时scandir一次需要约10s
* 其次，分析原逻辑2得知：scandir十分频繁

### 优化

* 优化点1：采用JDK自带的线程池ExecutorService = new ThreadPoolExecutor()（因任务队列大小需要自己定义）
  * 效果
    * 1、去除了runningFile来判断线程池isBusy的判断，省去了sychronized{runnigFile.add(cell)}的操作
    * 2、代码简洁、逻辑清晰
      * 省去了WorkerThread, taskList, runningFile,threadPool相关代码
* 优化点2：线程池大小设置为50，但是任务队列设置为5000，每次scanDir以后让线程池处理前5000个文件，处理完再继续下一次scanDir，再处理
  * 使用getActiveCount()，虽然返回值不一定可靠，但绝大部分情况还可以
  * 如果getActicveCount()逻辑等待5min还没有为0，也进行下一次遍历

### 限速

* 考虑到es的压力，需要限速
* 其实就是减少线程数和增大每个线程处理后的sleep时间

### 效果

* 从30min 3100个文件 => 290000个文件