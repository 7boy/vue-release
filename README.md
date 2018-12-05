> 关于一键打包发布Vue项目的方法有很多，这里介绍用nodejs + ssh2模块上传文件至服务器

- [源码点这里](http://note.youdao.com/)

### 步骤
1. 打包并压缩dist文件夹
2. 连接服务器
3. 清空原有文件
4. 上传压缩好的文件
5. 解压文件

### 打包并压缩dist文件夹
期望值是一行命令进行打包上传，所以在package.json中配置命令
```
"scripts": {
    "serve": "vue-cli-service serve",
    "build": "vue-cli-service build",
    "publish": "npm run build && node publish.js && npm run serve"
  },
```
- 这样当我们运行npm run publish 的时候就会自动打包项目，然后用nodejs 执行publish.js文件了

- 使用 `compressing` 压缩dist文件夹
```
const compressing = require('compressing');
// 压缩dist目录文件
function compress() {
  return new Promise((resolve, reject) => {
    compressing.tar.compressDir('./dist', 'dist.tar')
      .then(() => {
        resolve(true)
        console.log('压缩成功')
      })
      .catch(err => {
        reject(err)
        console.error(err);
      });
  })
}
```
该方法是把当前目录下的 `dist`文件夹 压缩成`dist.tar`放到当前目录下

### 连接服务器
- 使用 `ssh2` 模块连接服务器，并监听连接状态
```
// 服务器账号
const server = {
  host: '0.0.0.0',
  username: 'test',
  password: '123456',
  port: 2001
}
// 连接服务器
const Client = require('ssh2').Client
async function Connect  () {
  return new Promise((resolve, reject) => {
    conn.on("ready", function(){
      console.log('服务器连接成功')
      resolve(conn)
    }).on('error', function(err){
      console.log('服务器连接失败')
      reject(err)
    }).on('end', function() {
      console.log('服务器连接关闭')
    }).on('close', function(had_error){
    }).connect(server);
  })
}
let c = Connect()
```
- 这样就成功连上了服务器，接下来可以运行 `shell` 命令删除服务器上的文件

### 清空原有文件
- 向服务器发送 `shell` 指令 删除原项目文件
```
// 执行shell脚本
async function Shell (c,cmd){
  return new Promise((resolve, reject) => {
    c.exec(cmd,function(err, stream) {
      if (err) { reject(err) }
      stream.on('close', function() {
        resolve(c)
      }).on('data', function(data) {

      }).stderr.on('data', function(data) {
        console.log('shell执行失败')
      });
    });
  })
}
// 这里的 c 为服务器的连接对象
c = await Shell(c,`rm -rf demo/*`)
```

### 上传压缩好的文件
- 将上面压缩好的dist文件夹传入我们项目的目录
```
// 上传文件
function UploadFile(c){
  console.log('开始上传')
  return new Promise((resolve, reject) => {
    c.sftp(function (err, sftp) {
      if (err) {reject(err);}
      else {
        // remotePath 上传的服务器路径
        sftp.fastPut('./dist.tar', remotePath , function (err, result) {
          if(err){reject(err)}
          resolve(c)
        });
      }
    });
  })
}
```
- 这样压缩好的 `dist.tar` 就被上传至服务器了

### 解压文件
- 将服务器上的 `dist.tar` 解压出来就大功告成
- 这里还是让服务器执行 `shell` 脚本
```
const shellList = [
    `cd /demo \n`,
    `tar xvf dist.tar \n `,
    `mv dist/* ./  \n`,
  ]
  c = await Shell(c,shellList.join(''))
```
- 到这里运行 `npm run publish` 就可以一键发布项目了

### 附
- 这里只提供了一个最简单的方法，其实还有很多待完善的

1.不把服务器密码放在项目中(可以用readline)
2.备份旧的压缩包
3. ...

- [源码点这里](http://note.youdao.com/) 
