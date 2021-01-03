## 创建一个可以用于qiankun项目中的Angular微应用

> 基于业务需要，搭建一个能够独立运行也可以在qiankun项目中作为微应用嵌入的Angular项目的过程记录

### 环境安装

Node.js和npm的安装不赘述。

因为tslint已经停止继续更新，而Angular-cli默认仍使用tslint作为代码规范检查工具，我们应该改为使用[angular-eslint](https://github.com/angular-eslint/angular-eslint)



1. 新建一个空文件夹并切换到该文件夹下

2. 安装环境（默认Angular10.1以上，angular-eslint支持最低10.1）

    ```sh
        # 安装Angular/cli和angular-eslint的schematics
        npm i -g @angular/cli @angular-devkit/{core,schematics} @angular-eslint/schematics
        
        # 使用这些schematics建立一个用angular-eslint检查代码的angular项目
        ng new --collection=@angular-eslint/schematics
    ```
