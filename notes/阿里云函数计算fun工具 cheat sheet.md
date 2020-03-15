# 阿里云函数计算fun工具 cheat sheet

## 基本使用

### 安装fun

```shell
$ npm install @alicloud/fun -g
```

### 初始化工程

新建工程目录，cd进入该目录，执行如下语句

```shell
$ fun init 
```

### 设置配置参数

```shell
$ fun config
```

### 发布函数

```shell
$ fun deploy
```

## 依赖安装

## 使用配置文件安装

### 生成fun.yml配置文件

```shell
$ fun install init
```

### 配置文件

```yml
runtime: python3
tasks:
  - name: install psycopg2
    pip: psycopg2
    local: false
  - name: install xlsxwriter
    pip: xlsxwriter
    local: false
```

### 安装

```shell
$ sudo fun install
```

## 使用pip直接安装

```shell
$ sudo fun install --runtime python3 --package-type pip --save psycopg2
```

-- runtime指定运行时环境

--package-type指定安装依赖的类型，pip或apt

--save表示持久化，该安装被会写入到fun.yml配置文件中

## 依赖安装成功的现象

安装成功后会出现.fun文件夹

![1569569347308](/home/floyd/.config/Typora/typora-user-images/1569569347308.png)

安装成功时控制台显示如下

![1569569370215](/home/floyd/.config/Typora/typora-user-images/1569569370215.png)