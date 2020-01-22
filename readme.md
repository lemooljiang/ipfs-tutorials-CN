# <center>IPFS实用指南</center>

- [说明](#说明)
- [简介](#简介)
- [下载与资源](#下载与资源)
- [IPFS的公共网关](#ipfs的公共网关)
- [一、服务器端(用go版)](#一、服务器端(用go版))
- [安装](#安装)
- [查看配置config](#查看配置config)
- [设置端口和跨域](#设置端口和跨域)
- [systemctl-service](#systemctl-service)
- [nginx配置反向代理](#nginx配置反向代理)
- [二、前端(js-ipfs-http-client)](#二前端js-ipfs-http-client)
- [安装和配置](#安装和配置)
- [上传图片到ipfs](#上传图片到ipfs)
- [上传视频到ipfs](#上传视频到ipfs)
- [videostream播放视频](#videostream播放视频)


## 说明
用IPFS提供web服务，需要有两部分，一是服务器端开启daemon, 二是前端用js来访问服务器。这与一般的网站服务是一样的。所以，在此我分为服务器端和前端两部分来实现IPFS的服务。

## 简介
IPFS的全称是InterPlanetary File System（星际文件系统），从名称上看，这是一个很炫酷、很有野心的项目。简单地说它就是一个点对点的分布式文件系统。官网和github都可以找到所有的相关资料。建议从它的白皮书，和直译中文版本开始了解，后面我们会慢慢地认识它。白皮书上指出了多个应用场景。

## 下载与资源
[官网](https://ipfs.io)
[手册](https://docs.ipfs.io)

## IPFS的公共网关
IPFS的公共网关还是蛮稀缺的，而且很多在国内是访问不了的，所以，只能多试下啰。

[查询公共网关](https://ipfs.github.io/public-gateway-checker/)
```
https://ipfs.io/ipfs
https://ipfs.jeroendeneef.com/ipfs
https://na.siderus.io/ipfs/
https://ipfs.busy.org/ipfs
http://gateway.ipfs.io/ipfs
https://siderus.io/ipfs/
https://ipfs.jes.xxx/ipfs/
https://eu.siderus.io/ipfs/
https://na.siderus.io/ipfs/
https://ipfs.eternum.io/ipfs/
https://hardbin.com/ipfs/
https://gateway.temporal.cloud/ipfs/
https://jorropo.ovh/ipfs/
```


# 一、服务器端(用go版)

## 安装
```
wget https://github.com/ipfs/go-ipfs/releases/download/v0.4.22/go-ipfs_v0.4.22_linux-amd64.tar.gz

tar -zxvf go-ipfs_v0.4.22_linux-amd64.tar.gz
cd go-ipfs
mv ipfs /usr/local/bin/ipfs
ipfs version
ipfs --help  

//初始化，/root/.ipfs
ipfs init

ipfs id     //自己的id
ipfs daemon //开启进程
```

## 查看配置config
cd /root/.ipfs

[配置](./dates-win/go-ipfs-config.md)

## 设置端口和跨域
js版无法跨域，不要用，用go版
```
//需要改动的地方
"Addresses": {
    "API": "/ip4/127.0.0.1/tcp/9000",
    "Announce": [],
    "Gateway": "/ip4/127.0.0.1/tcp/8080",
"StorageMax": "80GB"

//在控制台粘入以下6条命令
ipfs config Addresses.Gateway /ip4/127.0.0.1/tcp/8080  //访问接口
ipfs config Addresses.API /ip4/127.0.0.1/tcp/9000    //上传接口
ipfs config --json API.HTTPHeaders.Access-Control-Allow-Methods '["PUT","GET", "POST", "OPTIONS"]'
ipfs config --json API.HTTPHeaders.Access-Control-Allow-Origin '["*"]'
ipfs config --json API.HTTPHeaders.Access-Control-Allow-Credentials '["true"]'
ipfs config --json API.HTTPHeaders.Access-Control-Allow-Headers '["Authorization"]'
ipfs config --json API.HTTPHeaders.Access-Control-Expose-Headers '["Location"]'
```

## systemctl-service
用systemctl管理ipfs的进程和重启等事务
```
which命令
查看ipfs执行文件的位置:
/usr/local/bin/ipfs

//创建文件
vim /lib/systemd/system/ipfs.service

//粘入代码
[Unit]
Description=JSIPFS Daemon Client
[Service]
ExecStart=/usr/local/bin/ipfs daemon
Restart=always
RestartSec=30
Type=simple
User=root
Group=root
[Install]
WantedBy=multi-user.target

//命令
systemctl daemon-reload
systemctl enable ipfs
开启服务: systemctl start ipfs
关闭服务: systemctl stop ipfs
查看状态: systemctl status ipfs  //q退出
查看进程：ps -aux | grep ipfs
```

## nginx配置反向代理
IPFS是不提供https服务的，所以，必须使用nginx的反向代理提供https！

前提：nginx的https已由Certbot配置好了，再加上以下代码即可。

[nginx example](./dates-win/ipfs-nginx-conf-example.md)


#  二、前端(js-ipfs-http-client)

## 安装和配置
```js
cnpm install ipfs-http-client --save 
//main.js
import ipfsClient from 'ipfs-http-client'
const ipfs = ipfsClient({ host: 'example.com', port: '9001', protocol: 'https' })

Vue.prototype.ipfs = ipfs  //挂载到全局
```

## 上传图片到ipfs
```js
<b-form-file placeholder="上传封面图(建议尺寸：400*300像素)" 
              browse-text="文件" v-model="file" 
              class="title" accept=".jpg, .jpeg, .png, .webp, .gif" >
</b-form-file>

function upImage(file){
  return new Promise(resolve => {
    const reader = new FileReader()
    reader.readAsArrayBuffer(file)
    reader.onload = (event) => {
      this.ipfs.add({
            path: file.name,
            content: Buffer.from(event.target.result)
          },
          (error, added) => {
            if (error) {
              console.log(123, error)
            } else {
              //上传成功
              console.log(234, added[0].hash)
              resolve(added[0].hash)
            }
          })
    }
  })
}
```

## 上传视频到ipfs
和上传图片是一个函数，在细节上略有不同。
```js
<b-form-file placeholder="上传mp4视频" browse-text="文件" v-model="file" class="mb-2" accept=".mp4" ></b-form-file>

//上传视频
let file = this.file
const reader = new FileReader()
reader.readAsArrayBuffer(file)
reader.onload = (event) => {
  that.ipfs.add({
    path: file.name,
    content: Buffer.from(event.target.result)},
  (error, added) => {
    if (error) {
        console.log(123, error)
    } else {
        //上传成功
        that.addNew = true
        let body = added[0].hash
        console.log(234, body)
    }
  })
}
```

## videostream播放视频
```js
cnpm install videostream --save

<video ref="myvideo" controls autoplay="autoplay" width="90%" ></video>

import VideoStream from 'videostream'
Play(hash){
  //得到视频hash播放
  let that = this
  let stream
  new VideoStream({
    createReadStream: function createReadStream (opts) {
      const start = opts.start
      const end = opts.end ? start + opts.end + 1 : undefined
      // If we've streamed before, clean up the existing stream
      if (stream && stream.destroy) {
        stream.destroy()
      }
      // This stream will contain the requested bytes
      stream = that.ipfs.catReadableStream(hash,  {
        offset: start,
        length: end && end - start})
      stream.on('error', (error) => console.log(error))
      return stream
    }
  }, that.$refs.myvideo)
},
```
