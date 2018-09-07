本文档以调试zun-compute服务为例
devstack环境下各项目的代码在/opt/stack/目录下，在zun目录下找到想要debug的代码文件，将
```python
import pdb;pdb.set_trace()
```
插入到待调试的代码之前（标记断点），保存文件后关闭zun-compute服务
```bash
sudo systemctl stop devstack@zun-compute
```
然后执行
```bash
/bin/zun-compute
```
在命令行状态下启动服务，当程序执行到你添加断点的地方时会停下来，屏幕右下角会显示（pdb），在后边输入相应的指令即可对代码进行一步步的调试

附：pdb指令

* b 设置断点
* c 继续执行程序, 或是跳到下个断点
* l 查看当前行的代码段
* s 进入函数
* r 执行代码直到从当前函数返回
* q 中止并退出
* n 执行下一行
* p 打印变量的值，例如p a表示打印a的值