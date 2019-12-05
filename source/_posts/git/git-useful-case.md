# Git 使用

本文介绍一些常用的、基础的git使用姿势，以及一些在特定场景中会用到的命令

## 基础

这里是一些日常最常用的使用方式

### git clone 

克隆git上代码到本地

### git checkout

检出分支或带个文件，此命令可用来放弃还未add到缓存区的文件

### git pull

拉取代码，将git仓库代码拉取到本地

### git add

将文件添加到缓冲区

### git commit

提交文件到本地仓库

### git push

将已commit的文件推送到远端仓库

### git revert 

将已经推送到远端仓库的提交还原

### git format-patch

将本地当前的修改添加到一个patch文件，新增patch文件成功后，本地分支代码将会被恢复到修改前的初始状态 

## 进阶

### merge

分支合并，常常用来将branch代码合并到master

### rebase

分支合并，常常用来将master代码合并到分支，当前分支的提交记录会追加在主干上

### reset

回退commit但尚未push的代码（如果代码已经commit，只能使用revert还原了）

reset分为3个操作类型

* hard
  
  完全还原到初始版本，本次提交中的所有变动将会丢失

* soft

  提交文件的变动部分不会丢失，变动的部分将会被保留在暂存区

* mixed
  
  提交文件的变动部分不会丢失，变动的部分将不会被保留在暂存区

### cherry-pick

挑樱桃，意为挑选好的东西出来。对于任意分支的任意的一次提交，可以单独将此次提交合并到当前分支

## 场景示例

> 注意：所有操作基于（但不仅限于）IDEA下
>
> 对于IDEA，文件在不同的状态下有不同的颜色，此处以原始主题为例
>
> 1. 未暂存：红色
> 2. 已暂存：新增文件为绿色，修改文件为蓝色，删除文件为灰色
> 3. 提交：白色
>
> 下列示例使用上述颜色描述

### 添加到暂存区的文件，如何移出暂存区

此操作主要针对新增的文件，在idea左下角功能菜单的`Version control`面板中，<font color='red'>右键绿色的文件 -> Local Changes -> Revert</font>

### commit后还未push，想撤销提交

此操作主要针对新增的文件，在idea左下角功能菜单的`Version control`面板中，<font color='red'>右键绿色的文件 -> Log</font>

比如撤销的提交为`ccccccc`，则如下图

![reset](./reset.png)

在回退框中，对应几种操作类型，对应到上述`reset`命令的描述

![reset-choose](./reset-choose.png)

### commit后已经push，想撤销提交

此操作主要针对新增的文件，在idea左下角功能菜单的`Version control`面板中，<font color='red'>右键绿色的文件 -> Log</font>

右键已经提交的记录，revert，重新push到远端

### 需要将分支的变动代码合并到主干

1. 切换到master分支
2. merge合并

   ![branch2master](./branch2master.jpg)

3. push

### 需要将主干代码合并到分支

1. 切换到branch分支
2. rebase合并

   ![rebase](./rebase.png)

3. push

### 不同分支并行开发

当前可能遇到在分支A开发工作做到一半，临时需要去分支B处理紧急事务，这时候可以将A分支所有变动代码保存到patch中，再将分支切换到B中进行开发，当B的工作开发完成后，切回A分支，再将patch的内容还原到分支上

操作过程

1. 右键项目 -> git -> repository 
2. 创建patch: stash changes
3. 从patch中恢复：unstash changes

### 开发过程中发现代码分支不正确，需要将修改移到正确分支

同上

### 使用IDEA合并代码后，还未push合并信息到远端仓库，需要回退本次merge的

* 使用reset命令回退（推荐）
  log面板中，在当前分支最后一次提交记录上，`右键 -> reset current branch to here`，选择`Hard`
* 使用rebase命令回退
  `右键项目 -> git -> repository -> rebase`，对话框中选择`from`属性为当前本地分支信息

### 使用idea合并代码后，已经push到远端仓库，需要回退本次merge的

> 此操作比较复杂，合并操作请审慎

假设有master和A分支，A分支内容已经被merge到master上，master现在需要将代码回退到merge前的状态

1. 切换到master分支
2. 使用reset回退本地代码到merge前的状态
3. 将本地代码生成一个新的分支`rallback/2019xxxx`，并checkout，将该新分支推送到远端仓库
4. 修改项目设置中的默认分支为：`rallback/2019xxxx`
5. 删除`master`分支
6. 使用当前的`rallback/2019xxxx`分支重新生成master分支，并checkout，将该新分支推送到远端仓库
7. 修改项目默认分支为：`master`
8. 删除`rallback/2019xxxx`分支











