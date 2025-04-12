# linux发行版之间的包名映射使用手册

## 主要流程

最终是从数据库取出软件包的信息进行比较，所以需要先把数据都录入数据库，主要流程如下所示：
* 源码包和二进制包下载与解压
* 提取源码包和二进制包的特征信息
* 将源码包和二进制包的特征信息入库
* 源码包比较得出源码包映射关系
* 二进制包比较得出二进制包映射关系

比较合适的数据库为Elastic Search，如果同学们有其他合适的选择也可以替换。

## 软件包入库

### 下载源码包、二进制包
在centos、和openeuler中，源码包的存储文件为rpm文件和primary.xml文件，二进制包的存储文件为filelists.xml文件。

| OS | 下载脚本 | 存储路径 |
| :----: | :---- | :---- | 
| centos | get_package/download_centos_7_src_rpm.sh | /your/store/path/ |
| openeuler | get_package/download_openeuler_openEuler-22.03-LTS_src_rpm.sh | /your/store/path/ |

### 解压源码包、二进制包

由于OS从镜像源中下载的源码包、二进制包是压缩文件，因此输入
>bash unzip_package/unzip_XXX.sh

| OS | 解压脚本 |
| :----: | :---- |
| centos | unzip_package/unzip_centos_7.sh |
| openeuler | unzip_package/unzip_openeuler_openEuler-22.03-LTS.sh |

## 提取特征

从下载的源码包和二进制包中提取出后续映射所需要的特征信息，并将这些特征信息存储在对应的json文件中。

* 源码包的特征信息：
> {
  "name": "",
  "version": "",
  "summary": "",
  "description": "",
  "url": "",
  "requires": [],
  "providers": [],
  "binaryList": [],
  "buildRequires": [],
  "source0": "",
  "macro_names": [],
  "email_names": [],
  "class_names": [],
  "path_names": [],
  "url_names": []
}
* 二进制包的特征信息：
> {
  "name": "",
  "arch": "",
  "filelist": [],
}

### 源码包特征提取

基于上述所介绍的不同OS存储源码包和二进制包信息的文件格式不完全相同，提取源码包特征的操作可分为两类：一类是需要从rpm包和primary.xml文件中提取特征，另一类是只需要从其源码包中提取。

对于centos、fedora、opensuse和openeuler，源码包的特征提取需要输入
> python feature_extract/extract_rpm.py -o "OS的名字"

命令来生成“XXX_rpm.json”文件，输入

> python feature_extract/extract_spec.py -o "OS的名称"

命令来生成“XXX_spec.json”文件。

之后，通过输入
> python utils/merge_json.py -rpm "XXX_rpm.json的文件路径" -spec "XXX_spec.json的文件路径" -merge "合并后的json的文件路径"

命令来将两个json文件整合在同一个json文件中。

### 二进制包特征提取
由于部分OS的二进制包中没有“filelist”特征信息，因此提取源码包特征的操作可分为两类：一类是从下载的二进制包中直接提取特征，另一类是需要通过“apt-file list”命令方式提取特征。

对于centos和openeuler，二进制包的特征提取通过输入
> python feature_extract/extract_bin.py -o "OS的名称"

命令来生成“XXX_binary.json”文件。

## 特征入库
该部分请同学们结合自己的存储方式实现。

## 软件包映射
该部分主要调用同学们实现的映射计算模块，例如可能以以下的命令实现：
> python package_mapping/pkg_mapping.py -o1 "{源OS名称}@{源OS版本号}" -o2 "{目标OS名称}@{目标OS版本号}

其中，源OS即为CentOS，目标OS即为openEuler。
## 结果查询
完成了上述的映射操作，通过输入
> cd /data/mapping

命令可以查询映射结果，所有的映射结果都会存储在该文件夹中，其中：
* /src文件夹中存放的yml文件是两个OS之间源码包的映射结果，其中带有“nosame”字段标题的yml文件为无同名软件包的映射结果，以“100”结尾标题的yml文件为随机抽取100条的映射结果；
* 除/src文件夹的其余文件夹中存放的yml文件是对应架构下，两个OS之间二进制包的映射结果，以“100”结尾标题的yml文件为随机抽取100条的映射结果。

当然，同学们可以不拘泥于上面的格式，以自己认为合适的形式存储并命名结果。

## 映射结果检验
该部分功能主要在interact目录中实现，同学们可以实现自己的检验逻辑，下面的为可以实现的功能。

由于二进制包只有name、arch等简单的特征，因此只检验源码包的映射结果，用于优化映射操作（调整权重参数、增加特征等）。通过执行

> python search_invalid.py -o1 "{源OS名称}@{源OS版本号}" -o2 "{目标OS名称}@{目标OS版本号}

命令来了解源码包映射结果情况，输出映射的准确率，并将无效的映射结果按照表格的形式存入"/analyse"文件夹中对应"invalid"开头的txt文件中。其中，源OS即为CentOS，目标OS即为openEuler。

此外，为了人工检验映射结果，可以通过执行

> python search_random.py -o1 "{源OS名称}@{源OS版本号}" -o2 "{目标OS名称}@{目标OS版本号}

命令从所有映射结果中随机抽取100条数据存入"/analyse"文件夹中对应"random"开头的txt文件中，其中，源OS即为CentOS，目标OS即为openEuler。

如果想要了解更为详细的全部映射结果，可以通过执行

> python search_all.py -o1 "{源OS名称}@{源OS版本号}" -o2 "{目标OS名称}@{目标OS版本号}

命令将全部的映射信息存入"/analyse"文件夹中对应"all"开头的txt文件中。其中，源OS即为CentOS，目标OS即为openEuler。
