# MinIO使用手册

### **1、在docker上部署minIO**

**①pull镜像**

~~~dockerfile
docker pull MinIO/MinIO

docker run -p 35793:9000 --name MinIO-test \

-d --restart=always \

-e "MinIO_ACCESS_KEY=cesc" \

-e "MinIO_SECRET_KEY=dywpt@2021" \

-v /filedisk1/soft/MinIO-test/data:/data \

-v /filedisk1/soft/MinIO-test/config:/root/.MinIO \

MinIO/MinIO server /data
~~~

参数说明:

- -p 35793:9000: 启动端口 , --name: MinIO-test docker中容器的名称

- -e "MinIO_ACCESS_KEY=cesc" : 连接账号 -e "MinIO_SECRET_KEY=dywpt@2021": 密码

- -v /filedisk1/soft/MinIO-test/data:/data:  文件存放文件夹映射 /data对应下面启动的/data

- -v /filedisk1/soft/MinIO-test/config:/root/.MinIO  MinIO:的配置文件映射

- server /data :/data指的是文件存放路径，可以挂接多个盘 格式为 /data1 /data2 /data3

- 注意的是多盘挂接主要是为了纠错和恢复(某块硬盘读取失败后，可以在别的盘读取，和写入)，多盘会保持同步数据

- MinIO单机部署多个数据盘存储的情况下，盘数一定要在4块以上，16块盘以下（受限于纠删码）

- 启动成功之后可以通过访问 ip:port 进入管理页面

  

### **2、Nginx中修改配置文件**

\- 访问MinIO页面相关的需要加上MinIO,示例:

~~~nginx
location /MinIO/{

  			proxy_pass http://localhost:35793/MinIO/;
   			proxy_next_upstream off;

  		}
~~~



**直接访问文件则不需要MinIO,示例:**

~~~nginx
location /file/{
                            proxy_set_header X-Real-IP $remote_addr;
                            proxy_set_header X-Forwarded-For                 $proxy_add_x_forwarded_for;
                            proxy_set_header X-Forwarded-Proto $scheme;
                            proxy_set_header Host $http_host;
     
                            proxy_connect_timeout 300;
                            # Default is HTTP/1, keepalive is only enabled in HTTP/1.1
                            proxy_http_version 1.1;
                            proxy_set_header Connection "";
                            chunked_transfer_encoding off;
     
     			proxy_pass http://localhost:35793/;
     			proxy_next_upstream off;
     		}
~~~



\- 文档介绍:

  \- http://docs.MinIO.org.cn/docs/master/setup-nginx-proxy-with-MinIO



### **3、部署MC（看具体需要）**

MC是一个可以通过命令行操作MinIO的工具

命令：

~~~dockerfile
docker pull minio/mc

docker run minio/mc ls play
~~~

mc就像是一个redis client那样的东西，通过docker安装每次都创建一个容器，交互还不是很方便。

上面第二条命令是直接查看默认的minio官网的一个OBS。

可以不使用docker安装，直接下载mc文件添加执行权限就可以使用了

目前大部分都可以用web端进行操作，除了同步数据以外



### **4、配置mc云服务（看具体需要）**

配置mc的云服务，也叫添加一个云存储服务，因为默认mc就带有一些去服务的配置，比如play，比如gcs和local。使用命令：

**mc config host add <ALIAS> <YOUR-S3-ENDPOINT> <YOUR-ACCESS-KEY> <YOUR-SECRET-KEY> [--api API-SIGNATURE]**

**<ALIAS>**别名就是给你的云存储服务起了一个短点的外号。必须要的。就是替代http://127.0.0.1:9000这个地址

**<YOUR-S3-ENDPOINT>**就是你的云服务的地址，比如http://127.0.0.1:9000

**<YOUR-ACCESS-KEY> <YOUR-SECRET-KEY>**是你的云存储服务账号密码，在启动docker容器时指定的。

**API**签名是可选参数，默认情况下，它被设置为"S3v4"。

云服务的配置文件在/root/.ms/config.json文件中，里面就保存了已经配置的云服务，比如play。进行如下配置前后可以对比查看一下配置文件。

示例：

\#删除默认的play服务 root@kermit:~/.mc# mc config host remove play Removed `play` successfully. #可以直接add配置实现覆盖已有的配置  root@kermit:~/.mc# mc config host add local http://127.0.0.1:9000/ username password Added `local` successfully.

\#可以删除添加的配置 root@kermit:~/.mc# mc config host remove gcs Removed `gcs` successfully.



## **5、在web端创建属于自己产品线的桶**

进入minio的web端

找到右下角的添加标志进行创建桶

![img](https://raw.githubusercontent.com/jasonfong11/img-folder/main/clipboard.png)



## **6、项目代码中的修改**

- 项目中增加minIO的配置类

~~~java
@Configuration
public class UploadSystemConfig {

    @Value("${ewater.upload.minio.minioBasePath}")
    private String minioBasePath;

    @Value("${ewater.upload.minio.endpoint}")
    private String endpoint;

    @Value("${ewater.upload.minio.accessKey}")
    private String accessKey;

    @Value("${ewater.upload.minio.accessSecret}")
    private String accessSecret;

//    @Value("${upload.localfileBasePath}")
//    private String localfileBasePath;

    @Value("${uploadFileLocalPath}")
    private String uploadFileLocalPath;

    @Bean
    public UploadConfig uploadConfig() {
        DefaultMinioUploadConfig defaultMinioUploadConfig = new DefaultMinioUploadConfig();
        defaultMinioUploadConfig.setMinioBasePath(minioBasePath);
        defaultMinioUploadConfig.setEndpoint(endpoint);
        defaultMinioUploadConfig.setAccessKey(accessKey);
        defaultMinioUploadConfig.setAccessSecret(accessSecret);
        return defaultMinioUploadConfig;
    }

    @Bean
    public UploadFileSystem uploadFileSystem(UploadConfig uploadConfig) {
        if (uploadConfig instanceof MinioUploadConfig) {
            return new MinioUploadFileSystem((MinioUploadConfig) uploadConfig);
        } else {
            return new LocalUploadFileSystem((LocalFileUploadConfig) uploadConfig);
        }
    }

    @Bean
    public Boolean transferTo(@Autowired UploadFileToSysUploadFile uploadFileToSysUploadFile) throws IOException {
        final Path path = Paths.get(uploadFileLocalPath);
        Files.walkFileTree(path, uploadFileToSysUploadFile);
        return true;
    }
}
~~~

- 配置文件中增加相应的配置信息

~~~properties
#minIo桶的名称
ewater.upload.minio.minioBasePath=emg
#minIo的地址和端口
ewater.upload.minio.endpoint=http://172.168.84.12:35793
#minIo的acKey
ewater.upload.minio.accessKey=cesc
#minIo的acS
ewater.upload.minio.accessSecret=dywpt@2021

#上传文件本地文件目录
uploadFileLocalPath=E:\\Img
~~~

- #### **更新后需要修改及迁移的相关代码**

- Ewater-core2.3.9:整合了Ewater-upload0.0.3

- UploadFileService中涉及代码已全部改为新版api

- 涉及到**Breaking Change**：

  - 源UploadFile 改为SysUploadFile
  - 源bizType改为path
  - 源bizId改为SysUploadFileOwner
  - 任何直接使用UploadFileRepository的代码都会失效

- 数据迁移:

  - Ewater-upload中提供:UploadFileToSysUploadFile用作默认迁移工具

    - ```
      FileVisitResult fileVisitResult = super.visitFile(file, attrs);
      String name = file.getFileName().toString();
      String fileNameNoEx = FilePathUtil.getFileNameNoEx(name);
      UploadFile uploadFile = (UploadFile)this.uploadFileRepository.findById(DataConvertUtil.strToLong(fileNameNoEx)).orElse((Object)null);
      if (uploadFile != null) {
          long id = uploadFile.getId();
          String bizType = uploadFile.getBizType();
          String sourceFileName = uploadFile.getSourceFileName();
          String bizId = uploadFile.getBizId();
          UploadSysFile uploadSysFile = UploadSysFile.builder().id(id).path(bizType).fileExt(uploadFile.getFileExt()).sourceName(sourceFileName).ownerId(bizId).fileReader(new LocalFileReader(file)).build();
          this.sysUploadFileAggregate.uploadFile(uploadSysFile);
      }
      ```

    - 本身继承了SimpleFileVisitor,指定了上传文件夹之后，会遍历文件夹中的所有文件，将其上传到系统中

    - ```
      //直接使用,只需要执行一次就可以
      final Path path = Paths.get(uploadFileLocalPath);
      Files.walkFileTree(path, uploadFileToSysUploadFile);
      ```



### 上传文件接口数据结构的改变

![image-20210727120449826](https://raw.githubusercontent.com/jasonfong11/img-folder/main/image-20210727120449826.png)



### 根据快速开始编写的测试接口

~~~java
//测试minIO上传功能
    public String inputMinIoTest(MultipartFile file) {
        String log = "";
        
        //构造sysUploadFile
        SysUploadFile sysUploadFile = new SysUploadFile();
        //minio上的文件地址
        String path = "test";
        sysUploadFile.setPath(path);
        //源文件名
        sysUploadFile.setSourceFileName(file.getOriginalFilename());
        //文件后缀名
        String fileExt = FilePathUtil.getExtensionName(file.getOriginalFilename());
        sysUploadFile.setFileExt(fileExt);

        //组合形成UploadSysFile对象
        UploadSysFile uploadSysFile = new UploadSysFile(sysUploadFile, new MultipartFileReader(file));

        //使用SysUploadFileAggregate将其upload
        SysUploadFileAggregate sysUploadFileAggregate = new SysUploadFileAggregate(uploadFileSystem, sysUploadFileRepository);
        SysUploadFile sysUf = sysUploadFileAggregate.uploadFile(uploadSysFile);

        log = sysUf.getFilePath();
        return log;

    }
~~~

其中，构造的**uploadSysFile**对象中的**fileReader**有如下几种可供选择

![image-20210727141701234](https://raw.githubusercontent.com/jasonfong11/img-folder/main/image-20210727141701234.png)

可根据不同的类型来选择对应的fileReader

### 可供选择的方法

- 删除文件:

  - 目前支持根据文件名来删除文件

  - ```java
    Long fileName=..;
    sysUploadFileAggregate.delete(fileName);
    ```

- 查找文件描述:

  - ```java
    //前置准备
    sysUploadFileRepository.save(builder("a", "1"));
    sysUploadFileRepository.save(builder("a", "2"));
    sysUploadFileRepository.save(builder("b", "1"));
    sysUploadFileRepository.save(builder("b", "2"));
    final SysUploadFile sy = builder("c", "3");
    sy.setId(234);
    sysUploadFileRepository.save(sy);
    ```

  - findByPathsAndOwner:

    - ```java
      final HashSet<SysUploadFileOwner> sysUploadFileOwners = new HashSet<>();
      final SysUploadFileOwner builder = builder("1");
      sysUploadFileOwners.add(builder);
      final List<SysUploadFile> byPathsAndOwners = sysUploadFileAggregate.findByPathsAndOwner(List.of("a", "b", "c"), builder);
      //结果=2
      ```

  - findByPathsAndOwners：

    - ```java
      final HashSet<SysUploadFileOwner> sysUploadFileOwners = new HashSet<>();
      final SysUploadFileOwner builder = builder("1");
      sysUploadFileOwners.add(builder);
      sysUploadFileOwners.add(builder("2"));
      sysUploadFileOwners.add(builder("3"));
      final List<SysUploadFile> byPathsAndOwners = sysUploadFileAggregate.findByPathsAndOwners(List.of("a", "b", "c"), sysUploadFileOwners);
      //结果=5
      ```

  - findAllByNames:

    - ```java
      final List<SysUploadFile> sysUploadFiles = sysUploadFileRepository.findAll();
      final List<Long> names = sysUploadFiles
          .stream().map(t -> t.getId()).collect(Collectors.toList());
      final List<SysUploadFile> all = sysUploadFileAggregate.findAllByNames(names);
      //两者相等
      ```

  - findByNameOrFileId:

    - ```java
      final List<SysUploadFile> sysUploadFiles = sysUploadFileRepository.findAll();
      final SysUploadFile sysUploadFile = sysUploadFiles.stream().findFirst().orElseThrow();
      final String fileId = sysUploadFile.getFileId();
      final long name = sysUploadFile.getId();
      final SysUploadFile byNameOrFileId = sysUploadFileAggregate.findByNameOrFileId(name, fileId);
      //两者相等
      final SysUploadFile byName = sysUploadFileAggregate.findByNameOrFileId(name, "");
      //两者相等
      final SysUploadFile byFileId = sysUploadFileAggregate.findByNameOrFileId(null, fileId);
      //两者相等
      ```

  - findByPathAndOwner:

    - ```java
      final String path = "a";
      final SysUploadFileOwner sysUploadFileOwner = builder("1");
      List<SysUploadFile> sysUploadFiles = sysUploadFileAggregate.findByPathAndOwner(path, null);
      //结果=2
      sysUploadFileAggregate.findByPathAndOwner("", sysUploadFileOwner);
      //结果=2
      sysUploadFileAggregate.findByPathAndOwner(path, sysUploadFileOwner);
      //结果=1
      ```

- 读取文件的内容:

  - 当获取到文件描述符:SysUploadFiles,可以直接通过toSysFile获取文件内容

  - ```java
    final SysUploadFile sysUf =..;
    final SysFile sysUploadFileSysFile = sysUf.toSysFile();
    final byte[] bytes = sysUploadFileSysFile.readBytes();
    final InputStream reader = sysUploadFileSysFile.reader();
    ```

- 遍历上传的文件夹:

  - 遍历上传文件夹可以通过UploadFileSystem提供的api

  - ```java
    //获取上传文件夹的文件
    final List<SysFile> sysFiles = uploadFileSystem.listFiles();
    for (SysFile sysFile : sysFiles) {
        //文件是否文件夹
        if (sysFile.isDirectory()) {
            //获取文件夹中的内容
            final List<SysFile> secondDir = sysFile.listFiles();
        }
    }
    ```











可参考文献：

https://blog.csdn.net/weixin_43582499/article/details/113315387