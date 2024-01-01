# init

## 参考
[一文读懂 Golang init 函数执行顺序](https://cloud.tencent.com/developer/article/2138066)

## 特点
可选的，无入参和返回值，自动执行不能被调用，没有数量限制

## 调用顺序
### 同一包
#### 同一源文件
同一文件中有多个init函数，从上到下执行

#### 不同源文件
根据文件名的字典序确定执行顺序

### 不同包
#### 无依赖关系
按照`main.go`中import顺序分别调用包的init，最后再调用main包的init。如果没有被import，则包中的init不会执行
* 无依赖关系指包之间没有依赖关系，在main文件中被import，并使用占位符'_'接收：`import _ "proj/a"`

#### 有依赖关系
从依赖最底层的包一层层向上调用，例如：main > a > b > c，init顺序应为 c > b > a > main

## 项目初始化流程
包内：常量 > 变量 > init

整体：从依赖最底层的包逐层向上初始化

![流程](https://ask.qcloudimg.com/http-save/yehe-2609282/8dc648c05980d8d9ea9f9b4aa2bc88ac.png)

## 写过的bug
同一个包，主函数读取命令行参数确定全局变量 `isProd` 的值，而redis和pg客户端的初始化放在了各自的init函数中，导致全局变量的改变没有被init函数使用。（即使当读取参数的代码放在main.go的init中，字典序小于main的所有go文件中的init函数都不能感知到main.go中读取到的参数）