## macOS编译OpenJDK13

### 1. 获取源码

#### 方式一

```
# 需要安装hg，由于没有国内的cdn节点，clone时间很长
hg clone https://hg.openjdk.java.net/jdk/jdk13
```

#### 方式二

1. 访问https://hg.openjdk.java.net/jdk/jdk13/）
2. 点击左侧菜单中的"Browse"，显示源码根目录页面。
3. 点击左侧"zip"链接即可下载当前版本打包好的源码，下载完成后本地直接解压即可。 

###  2. 系统配置

![system-config](https://tva1.sinaimg.cn/large/006tNbRwly1garlqfkmpuj312s0pugpo.jpg)

### 3. 编译环境

1. Xcode11.3

   - Xcode提供了OpenJDK所需的CLang编译器以及Makefile中用到的其他外部命令
   - 通过AppStore安装

2. Boot JDK

   - OpenJDK由多个部分（HotSpot、JDK类库、JAXWS、JAXP……）构成，其中一部分（HotSpot）代码使用C、C++编写，而更多的代码则是使用Java语言来实现，因此编译这些Java代码就需要用到另一个编译期可用的JDK，官方称这个JDK为“BootstrapJDK”。

   - 编译OpenJDK13时，BootstrapJDK必须使用JDK12及之后的版本。

   - 这里安装是12.0.2 (build 12.0.2+10)
     - https://download.java.net/java/GA/jdk12.0.2/e482c34c86bd4bf8b56c0b35558996b9/10/GPL/openjdk-12.0.2_osx-x64_bin.tar.gz
     - 解压设置下环境变量即可

### 4. 进行编译

1. Run configure

   ```
   # 具体参数含义可以使用"bash configure --help"查看
   bash configure --enable-debug --with-jvm-variants=server --enable-dtrace
   ```

   - 执行成功：

   ![](https://tva1.sinaimg.cn/large/006tNbRwly1garmsropqrj323b0u0ndb.jpg)

   - 若执行失败，请按照提示安装缺失的依赖。

   - configure命令承担了依赖项检查、参数配置和构建输出目录结构等多项职责，如果编译过程中需要的工具链或者依赖项有缺失，命令执行后将会得到明确的提示，并且给出该依赖的安装命令。

2. Run make

   ```
   # 按照当前的系统配置，6分钟make完毕
   make images
   ```

   - 执行完毕

   ![](https://tva1.sinaimg.cn/large/006tNbRwly1garn0bznpij31fo0mugvo.jpg)

3. 验证

   ```
   cd ~/jdk13/build/macosx-x86_64-server-fastdebug/jdk/bin
   ./java -version
   ```

   ![](https://tva1.sinaimg.cn/large/006tNbRwly1garncscsx9j31im068act.jpg)

### 5. 参考

1. http://hg.openjdk.java.net/jdk/jdk/raw-file/tip/doc/building.html
2. 周大大《深入理解Java虚拟机（第三版）》