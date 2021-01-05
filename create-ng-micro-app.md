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

3. 构建符合qiankun要求的微应用
   1. 在 src 目录新增 public-path.js 文件，内容为：
      ````javascript
         if (window.__POWERED_BY_QIANKUN__) {
           // eslint-disable-next-line no-undef
           __webpack_public_path__ = window.__INJECTED_PUBLIC_PATH_BY_QIANKUN__;
         }
      ````
   2. 设置 history 模式路由的 base，src/app/app-routing.module.ts 文件：
      ````typescript
         + import { APP_BASE_HREF } from '@angular/common';
      
         @NgModule({
         imports: [RouterModule.forRoot(routes)],
         exports: [RouterModule],
         // @ts-ignore
         +  providers: [{ provide: APP_BASE_HREF, useValue: window.__POWERED_BY_QIANKUN__ ? '/app-angular' : '/' }]
            })
      ````
   3. 修改入口文件，src/main.ts 文件。
      ````typescript
         import './public-path';
         import { enableProdMode, NgModuleRef } from '@angular/core';
         import { platformBrowserDynamic } from '@angular/platform-browser-dynamic';
         import { AppModule } from './app/app.module';
         import { environment } from './environments/environment';
         if (environment.production) {
            enableProdMode();
         }
         let app: void | NgModuleRef<AppModule>;
         async function render() {
            app = await platformBrowserDynamic()
            .bootstrapModule(AppModule)
            .catch(err => console.error(err));
         }
         if (!(window as any).__POWERED_BY_QIANKUN__) {
            render();
         }
         export async function bootstrap (props: Object) {
            console.log(props);
         }
         export async function mount (props: Object) {
            render();
         }
         export async function unmount (props: Object) {
            console.log(props);
            // @ts-ignore
            app.destroy();
         }
      
   4. 修改 webpack 打包配置

      先安装 @angular-builders/custom-webpack 插件
      
      ```shell
        npm i @angular-builders/custom-webpack@9.2.0 -D
      ```

      在根目录增加 custom-webpack.config.js ，内容为：
      ````javascript
         const appName = require('./package.json').name;
         module.exports = {
            devServer: {
            headers: {
               'Access-Control-Allow-Origin': '*',
               },
            },
         output: {
            library: `${appName}-[name]`,
            libraryTarget: 'umd',
            jsonpFunction: `webpackJsonp_${appName}`,
            },
         };
      ````
      修改 angular.json，将 [packageName] > architect > build > builder 和 [packageName] > architect > serve > builder 的值改为我们安装的插件，
      将我们的打包配置文件加入到 [packageName] > architect > build > options。
      ```json
         - "builder": "@angular-devkit/build-angular:browser",
         + "builder": "@angular-builders/custom-webpack:browser",
           "options": {
         +    "customWebpackConfig": {
         +      "path": "./custom-webpack.config.js"
         +    }
          }
      ```
      ```json
         - "builder": "@angular-devkit/build-angular:dev-server",
         + "builder": "@angular-builders/custom-webpack:dev-server",
      ```
   5. 解决zone.js问题

      为了不多次引入zone.js，将微应用的 src/polyfills.ts 里面的引入 zone.js 代码删掉。
      * 作为微应用
        
         在父应用引入 zone.js，需要在 import qiankun 之前引入。
   
      * 作为独立应用
      
         修改 angular.json将zone.js 作为assets打包进每次构建之后载入
         ```json
         // angular.json
        
        {
          "architect": {
           "build": {
            "assets": [
            +    {
            +       "glob": "zone.js",
            +       "input": "./node_modules/zone.js/dist/",
            +       "output": "/"
            +     }
              ]
            },
          
          }
        }
         ```
      在微应用的 src/index.html 里面的 <head> 标签加上内容，微应用独立访问时使用。
         
      ```html
      <script src="/assets/zone.js" ignore></script>
      ```
   6. 修正 ng build 打包报错问题

      修改 tsconfig.json 文件
   
      ```json
         - "target": "es2015",
         + "target": "es5",
         + "typeRoots": [
         +   "node_modules/@types"
         + ],
      ```
   7. 为了防止主应用或其他微应用也为 angular 时，<app-root></app-root> 会冲突的问题，建议给<app-root> 加上一个唯一的 id，比如说当前应用名称。
      <br/><br/>
      src/index.html
      ```html
      - <app-root></app-root>
      + <app-root id="angular9"></app-root>
      ```
      <br/><br/>
      src/app/app.component.ts
      ```ts
      - selector: 'app-root',
      + selector: '#angular9 app-root',
      ```

      
### 到此，一个可以独立使用也可以作为微应用载入的Angular项目就构建完成
