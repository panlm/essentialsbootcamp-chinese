.. _calm_single:

---------------------
Calm: 单虚拟机蓝图的创建
---------------------

.. note::

  完成本实验的估计时间为30分钟。

简介
++++

创建单虚拟机蓝图，并安装mysql 数据库，使用变量初始化数据库用户口令

准备工作
++++++++

提前准备CentOS镜像。

- 内网：`Download <http://10.42.194.11/images/1-Click-Demo/CentOS-7-x86_64-GenericCloud.qcow2>`_
- 外网：`Download <http://download.nutanix.com/calm/CentOS-7-x86_64-GenericCloud-1801-01.qcow2>`_

创建蓝图
++++++++

#. 选择 :fa:`bars` **> Services --> Calm**，进入Nutanix Calm界面

#. 在左手工具栏选择 |blueprints| **Blueprints** 查看和管理 Calm 蓝图。

#. 选择**创建** --> **单个虚拟机蓝图**

#. 第1步

  - **Name** - 输入蓝图名称
  - **Description (Optional)** - 输入蓝图对应的描述信息
  - **Project** - 指定蓝图所在的项目
  - **VM Detail** - 完成后点击进入下一步配置

  .. figure:: images/1.png

#. 第2步

  - **Name** - 输入虚拟机名称（在蓝图中的名称）
  - **Cloud** - 输入虚拟机运行的提供商名称
  - **Operation System** - 选择虚拟机操作系统类型
  - **VM Configuration** - 完成后点击进入下一步配置

  .. figure:: images/2.png

#. 第3步

  - **Name** - 输入虚拟机名称（在虚拟化中的名称）
  - **vCPUs** - 指定分配给虚拟机的虚拟CPU数量
  - **Cores per vCPU** - 指定每个虚拟CPU包含的核心数量
  - **Memory (GiB)** - 指定分配给虚拟机的内存数量
  - **Guest Customization** - 选中客户机定制化选项，以便初始化操作系统的用户名口令等相关基本信息

  ..

    #cloud-config
    disable_root: False
    ssh_pwauth: True
    password: this_is_a_password_for_default_user_centos
    chpasswd: { expire: False }

  .. figure:: images/31.png

  - **Operation** - 选择“从镜像服务克隆”，
  - **Image** - 选择操作系统镜像模板。（我们在<准备工作>章节中已下载该模板）

  .. figure:: images/32.png

  - **NICs** - 点击右侧蓝色加号，添加一个网卡，并选择对应的网络（网络需要能提供dhcp，或者IPAM的能力）
  - **SAVE** - 点击右下角蓝色保存按钮，进入下一步配置

  .. figure:: images/33.png

#. 第4步，选择”Advanced Options“，进入高级配置

  选择”Add/Edit Credentials“，添加一个用户。该用户可以登录虚拟机执行蓝图中定义的各种任务。（我们在初始化虚拟机时为该用户定义了口令，参照**第3步**）

  - **Add Credentials** - 点击左侧蓝色加号，添加账号
  - **Credential Name** - 在蓝图中显示的账号名称
  - **Username** - 对应操作系统中的用户名，centos是该模板中默认的用户名
  - **Secret Type** - 选择密码类型，password或者ssh private key
  - **Password** - 对应该用户的密码，在第3步中，我们设置该密码为**this_is_a_password_for_default_user_centos**

  .. figure:: images/41.png

  添加完第一个账号后，如下图所示，我们选择用该账号登录虚拟机。在**Connection**章节中，选中”**Check log-in upon create**“，并在**Credential**下选择刚才创建的用户名

  .. figure:: images/42.png

  **PreCreate**和**PostDelete**分别是指在虚拟机创建之前和在虚拟机销毁之后需要执行的任务，比如在一个未提供dhcp的环境中，需要在创建虚拟机之前通过类似IPAM的工具进行IP地址申请，然后在虚拟机资源被释放之后，释放占用的IP地址资源等，此时就需要用到该任务。本次实验环境中包含类似DHCP的功能，因此不需要配置该任务。

  .. figure:: images/43.png

  **Package Install**包含了虚拟机开机之后首次需要执行的任务，可以将对虚拟机进行初始化安装等工作配置在该任务中。
  
  **Package Uninstall**包含了虚拟机销毁之前需要执行的任务，可以将对虚拟机进行数据清理等工作配置在该任务中。

  .. figure:: images/44.png

  点击**Package Install**右侧的**Edit**按钮，开始配置任务。

  .. figure:: images/45.png

  我们按照以下步骤创建一个简单的任务来安装mysql数据库软件

  - **Add Task** - 点击添加新任务，默认名为**Task1**。选中**Task1**进行配置。
  - **Task Name** - 设置任务名称为**Install mysql package**
  - **Type** - 选择任务类型为**Execute**
  - **Script Type** - 选择脚本类型为**Shell**
  - **Endpoint (Optional)** - 留空
  - **Credential** - 选择之前添加的用户名
  - **Script** - 复制粘贴下面代码

  .. code-block:: bash

    #!/bin/bash
    set -x

    mysql_password="@@{DB_PASSWORD}@@" ## HERE is a variable in Calm

    sudo yum -q install -y epel-release
    sudo yum -q install -y wget git python3-pip python-virtualenv gcc python3-devel bc lvm2

    ## install mysql
    sudo yum install -y --quiet "http://repo.mysql.com/mysql57-community-release-el7.rpm"
    sudo yum install -y --quiet sshpass mysql-community-server mysql-community-devel
    sudo systemctl enable mysqld
    sudo systemctl start mysqld
    ## Fix to obtain temp password and set it to blank
    ## for mysql 5.7
    password=$(sudo grep -oP 'temporary password(.*): \K(\S+)' /var/log/mysqld.log |tail -n 1)
    sudo mysqladmin --user=root --password="$password" password aaBB**cc1122
    sudo mysql --user=root --password=aaBB**cc1122 -e "UNINSTALL PLUGIN validate_password"
    sudo mysqladmin --user=root --password="aaBB**cc1122" password "${mysql_password}"


  除了上述初始化安装脚本之外，用户可以在添加自定义的其他任务。例如下图，我们可以添加一个mysql备份的任务，以便需要执行备份时，只需要简单点一下即可运行，不会引入人为错误。

  .. figure:: images/46.png

  点击**Add Action**打开任务编辑界面，并在页面左上角输入该Action名称

  .. figure:: images/47.png

  - **Add Task** - 点击添加新任务，默认名为**Task1**。选中**Task1**进行配置。
  - **Task Name** - 设置任务名称为**mysql backup**
  - **Type** - 选择任务类型为**Execute**
  - **Script Type** - 选择脚本类型为**Shell**
  - **Endpoint (Optional)** - 留空
  - **Credential** - 选择之前添加的用户名
  - **Script** - 复制粘贴下面代码

  .. code-block:: bash
  
    #!/bin/bash
    set -x

    ## Setup variables
    mysql_password="@@{DB_PASSWORD}@@" ## HERE is a variable in Calm
    dest="@@{BACKUP_FILE_PATH}@@"      ## HERE is a variable in Calm

    date_part=`date +%F`
    mkdir -p @@{BACKUP_FILE_PATH}@@
    sudo mysqldump -u root -p${mysql_password} --all-databases | sudo gzip -9 > ${dest}/db_dump.sql.gz  

  上面脚本中我们使用了两个自定义变量: DB_PASSWORD 和 BACKUP_FILE_PATH。接下来我们对这两个变量进行初始化配置。点击页面右上方**App Variables**。添加第一个变量 DB_PASSWORD

  - **Name** - 变量名称为**DB_PASSWORD**
  - **Data Type** - 选择变量类型为**String**
  - **Value** - 输入变量默认值
  - **Secret** - 选中该选项，则变量显示为秘钥字符串，以 * 代替

  .. figure:: images/51.png

  添加第二个变量 BACKUP_FILE_PATH

  - **Name** - 变量名称为**BACKUP_FILE_PATH**
  - **Data Type** - 选择变量类型为**String**
  - **Value** - 输入变量默认值**/tmp**

  .. figure:: images/52.png

  点击完成，返回蓝图窗口

  点击**Save** 保存蓝图

运行蓝图
++++++++
