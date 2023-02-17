# Salome_Meca Install



> 基于ubuntu 20.04



## 1.安装必要依赖包

```shell
#!/bin/bash
sudo apt install vim python -y
sudo apt install net-tools -y
sudo apt-get  install  build-essential -y
 
#OpenGL安装
sudo apt-get install libgl1-mesa-dev  -y
#OpenGL Library
sudo apt-get install libglu1-mesa-dev  -y

#OpenGL Utilities
sudo apt-get install freeglut3-dev  -y
#OpenGL Utility Toolkit
#如果OpenGL Utilities报错可尝试 sudo apt-get install libglut-dev  -y
 
sudo apt-get install libcanberra-gtk-module libcanberra-gtk3-module  -y
 
sudo apt-get install python3-dev -y
sudo apt-get install python3-numpy -y
sudo apt-get install tcl tk -y
sudo apt-get install bison -y
sudo apt-get install flex -y
sudo apt-get install liblapack-dev -y
sudo apt-get install libblas-dev -y
sudo apt-get install libopenblas-dev -y
sudo apt-get install libopenblas-base -y
 
sudo apt-get install net-tools -y
sudo apt-get install libnlopt0 -y
 
sudo apt-get install libx11-dev libxext-dev libxtst-dev libxrender-dev libxmu-dev libxmuu-dev -y
sudo apt-get install build-essential libgl1-mesa-dev libglu1-mesa-dev freeglut3-dev -y
```

> 注意：确保当前python=python2
>

## 2.下载

地址：https://code-aster.org/FICHIERS/salome_meca-2019.0.3-1-universal.tgz

```
# wget到~/salome目录
tar -zxvf salome_meca-2019.0.3-1-universal.tgz
cd salome_meca-2019.0.3-1-universal
chmod 755 salome_meca-2019.0.3-1-universal.run
./salome_meca-2019.0.3-1-universal.run -t /home/[username]/salome_meca -l English -D -f
```



## 3.启动



```
cd  /home/[username]/salome_meca/appli_V2019.0.3_universal
 ./salome # 即可运行 salome_meca gui图形界面
```



## 4.错误



> 此时切换到AsterStudy模块会报无法加载模块错误



修复方法：

```shell
cd [yourpath]/salome_meca/V2019.0.3_universal/prerequisites/debianForSalome/lib
rm libstdc++.so libstdc++.so.6 libstdc++.so.6.0
sudo ln -s /usr/lib/x86_64-linux-gnu/libffi.so.7 /usr/lib/x86_64-linux-gnu/libffi.so.6
```



## 5.Linux命令行运行Code_Aster用例



```shell
git clone https://github.com/Jesusbill/code-aster-examples
cd  code-aster-examples-master\Tutorial_13
```

创建脚本 run_aster.sh

```shell
#!/bin/bash 

export PATH=$PATH:/home/yypan/salome_meca/V2019.0.3_universal/tools/Code_aster_frontend-20190/bin
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/yypan/salome_meca/V2019.0.3_universal/prerequisites/debianForSalome/lib
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/yypan/salome_meca/V2019.0.3_universal/prerequisites/Mfront-TFEL321/lib
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/yypan/salome_meca/V2019.0.3_universal/prerequisites/Medfichier-400/lib
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/yypan/salome_meca/V2019.0.3_universal/prerequisites/Hdf5-1103/lib
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/home/yypan/salome_meca/V2019.0.3_universal/prerequisites/Python-365/lib

#ldd /home/yypan/salome_meca/V2019.0.3_universal/tools/Code_aster_stable-v144_smeca/bin/aster

fullComm=$(find . -name "*.comm")
nameComm=$(basename $fullComm)
echo $fullComm
echo $nameComm

fullMed=$(find . -name "*.med")
nameMed=$(basename $fullMed)
nameResu=$(basename $nameMed .med)
echo $fullMed
echo $nameMed


touch as.export
echo "" > as.export

echo "P actions make_etude" >> as.export
echo "P lang en" >> as.export
echo "P memjob 5120000" >> as.export
echo "P memory_limit 5000.0" >> as.export
echo "P mode interactif" >> as.export
echo "P mpi_nbcpu 1" >> as.export
echo "P mpi_nbnoeud 1" >> as.export
echo "P time_limit 900.0" >> as.export
echo "P tpsjob 16" >> as.export
echo "P version stable" >> as.export
echo "A memjeveux 625.0" >> as.export
echo "A tpmax 900.0" >> as.export 
echo "F comm $nameComm D  1" >>as.export
echo "F mmed $nameMed D  20" >>as.export
echo "F mess output.mess R  6" >>as.export
echo "F rmed $nameResu.rmed R  80" >>as.export
echo "F resu $nameResu.resu R  10" >>as.export


as_run --run as.export
```

然后执行 run_aster.sh 即可命令行运行code_aster用例



## 6.Window命令行运行Code_Aster用例



```shell
将as_run.bat设置环境变量PATH中

as_run.bat --run run.export
```