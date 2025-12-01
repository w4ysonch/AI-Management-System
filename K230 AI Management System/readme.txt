项目文件结构：
i2c_ssd1306.py-------------------------rtduino上oled驱动代码（包含于RT-Smart镜像内核源码中）
main.py---------------------------------主函数
mp_deployment_source--------------micropython部署资源（将训练网站上训练好的 AI 模型，转化为 K230 芯片的 NPU（KPU）可以直接运行的格式）
test2.zip--------------------------------模型训练部署zip包
readme.txt-----------------------------必读文件
项目代码介绍.txt-----------------------项目代码详解


      本项目使用的是RT-Thread Smart AI套件（淘宝链接：https://e.tb.cn/h.SBGMFrRTHpvHlVG?tk=OTVafpM7qii），芯片为嘉楠K230（开发板资料：https://www.kendryte.com/k230/zh/rtt/dev/index_rtt_only.html）；
      烧录的镜像为CanMV镜像文件(CanMV-K230_DONGSHANPI_micropython_local_nncase_v2.9.0.img.gz)，镜像文件下载：（https://pan.baidu.com/s/1os9wadhNvpo3ZObLgbENFQ#list/path=%2F&parentPath=%2F 密码：rtth），启动方式为SD卡启动(boot0-->off;boot1-->off)，环境为MicroPython；
      项目所用数据集为图像分类数据集，进行模型训练的网站为：https://www.kendryte.com/zh/training/start；
      训练完成的部署zip包为：test2.zip
配置运行方法：
      1. 直接将已经配置好的文件夹(mp_deployment_source)解压后复制到sdcard目录下；
      2. 将i2c_ssd1306.py、main.py复制到sdcard目录下；
      3. 打开main.py(记事本或pycharm)，复制代码，打开CanMV IDE K230软件粘贴代码并运行。


