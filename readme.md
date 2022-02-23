## 需求

打包下载 OSS 上存储的多个文件，例如将OSS上的一个目录打包下载。这样可以节省网络传输的数据，达到减少费用和下载时间的效果。

![](https://data-analysis.cn-shanghai.log.aliyuncs.com/logstores/article-logs/track_ua.gif?APIVersion=0.6.0&title=%E4%BD%BF%E7%94%A8%E5%87%BD%E6%95%B0%E8%AE%A1%E7%AE%97%E6%89%93%E5%8C%85%E4%B8%8B%E8%BD%BDOSS%E6%96%87%E4%BB%B6&author=rockuwu&src=article)

## 方案

使用函数计算先把多个文件压缩成一个 zip，存储到 OSS上面，返回 zip 文件的地址，客户端下载此文件。一般的客户端都支持跟随 HTTP 302 跳转地址，所以在完成压缩后，返回一个302的地址，客户端再跟随这个地址下载压缩后的文件包。

![zip_oss_high](https://img.alicdn.com/tfs/TB1GitkyeL2gK0jSZPhXXahvXXa-1258-946.png)


## 部署

### 准备工作

[免费开通函数计算](https://statistics.functioncompute.com/?title=%E4%BD%BF%E7%94%A8%E5%87%BD%E6%95%B0%E8%AE%A1%E7%AE%97%E6%89%93%E5%8C%85%E4%B8%8B%E8%BD%BDOSS%E6%96%87%E4%BB%B6&author=rockuwu&src=article&url=https%3A%2F%2Ffc.console.aliyuncs.com) 

[免费开通对象存储 OSS](https://oss.console.aliyun.com/)

### 1. clone 解压工程

```bash
git clone https://github.com/awesome-fc/zip-oss.git
```

如果没有安装 git， 可以直接在浏览器输入 `https://codeload.github.com/awesome-fc/zip-oss/zip/master` 下载代码 zip 包

### 2. 安装最新版本的 Serverless Devs Tool

- [安装 Serverless Devs Tool](https://help.aliyun.com/document_detail/195474.html)

- [配置](https://help.aliyun.com/document_detail/295894.html)

### 3. 部署函数， 执行 `s deploy`， 部署成功后如下图所示：

![](https://img.alicdn.com/imgextra/i3/O1CN015spfAO1wBi4Wp8ezs_!!6000000006270-2-tps-887-250.png)


### 4. 调用函数
1. 在OSS上准备要打包的文件
    - 把文件放在OSS上面一个目录下面， 比如 `files`目录

2. 触发函数(通过HTTP trigger地址)
    - 使用curl命令直接调用函数

```bash
cat <<EOF > event.json
{
  "region": "cn-shanghai",
  "bucket": "fc-test-tianlong-wu",
  "source-dir": "files/"
}
EOF

curl -v -L -o /tmp/my.zip -d @./event.json https://123456789.cn-shanghai.fc.aliyuncs.com/2016-08-15/proxy/zip-service/zip-oss/
```

打开`/tmp/my.zip`，就是`files/`目录下所有文件的压缩包。

## 实现细节

1. 函数运行环境的磁盘空间是有限的，采用流式下载和上传的方式，只在内存中缓存少量的数据。
2. 为了加快速度，一边生成zip文件时一边上传到OSS
3. 上传zip文件到OSS时，利用OSS分片上传的特性，多线程并发上传

![zip_oss_low](https://img.alicdn.com/tfs/TB13jVqyoY1gK0jSZFCXXcwqXXa-774-1066.png)

## 实验

### 实验数据

|#|文件数|压缩前总大小|压缩后总大小|执行时间|
|---|---|---|---|---|
|1|7|1.2MB|1.16MB|0.4s|
|2|57|1.06GB|1.06GB|63s|