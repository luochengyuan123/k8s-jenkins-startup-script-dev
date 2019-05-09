## jenkins安装初始化   
### 1、安装依赖插件   
在【系统管理】【插件管理】安装需要的插件   
```
Git Parameter Plug-In
Git plugin
Pipeline Utility Steps
SSH Pipeline Steps
SSH Slaves plugin
SSH Agent Plugin
```
### 2、配置maven  
在【系统管理】【全局工具配置】配置  
name: M3  
自动安装：打勾  
version: 3.6.0  

### 3、添加集成帐号  
在【凭据】【系统】【全局凭据】【添加凭据】添加  
gitlab的帐号   
username: only_read_dev  
password: only_read_dev  
ID: 692a8f27-c716-42c8-a334-81db86d891a5

harbor的帐号  
username: cicd  
password: Pwd@cicd2019  
ID: 9b041abc-e0e6-43ec-a87c-f65a74a125c8  
