# 实现一个可以在点击处弹出的popup组件的功能

> 基于业务需要，需要自己实现一个在鼠标点击处弹出一个popup组件然后点击选择上传方式并弹出弹窗进行操作的功能

## 业务要求

需要在B端(React框架)和C端（Vue框架）实现一个功能相同的上传组件，该上传组件具备以下功能：

- 点击指定区域（通常是上传按钮）时，弹出一个如图的popup弹窗，供用户选择上传方式
  <br/>   
  ![popup图](assets/images/popup.png)   
  <br/>
- 点击"素材库选择"时，打开一个网盘选择窗口，在其中选择某个符合业务要求的文件，并将文件信息返回给业务系统
- 点击"本地上传"时，先打开文件选择窗口，选择本地文件上传，并在上传完成之后选择是否要同步到素材库中，如果是单个图片就进行尺寸裁切（裁切比例按照业务系统传入的参数确定），裁切后进行上传再选择是否要同步到素材库
- 需要框架不同的两个端均可以实现相同功能，样式和逻辑完全一致

## 思考

在该功能之前，已经因为之前的需求实现了在指定容器中渲染一个网盘文件管理器（带有查看、上传、编辑、删除文件或文件夹功能）的react为基础的sdk，因此该次应考虑在该sdk基础上添加新的方法来实现该功能，同时这样也能保证多端的统一性和后续维护的简便性

## 实践

### 1、实现将react项目打入一个sdk中供人引用

该部分之前已经实现过，具体方式为将一个React项目的初始化部分作成一个构造器函数，
并在webpack打包配置中将该函数作为入口打包到单一文件中，页面通过载入该文件（可以通过js引入或npm的import方式），
将构造器函数挂载到window对象上，在需要使用时将容器元素和其他sdk内部要求的参数作为新建实例的参数实例化，
实例化过程中会自动将该React项目初始化在指定的容器上

入口文件例：

```jsx
 class SDKName {
  constructor(options) {
    this.options = options
    // ... 业务操作
    this._init()
  }
  
  _init() {
    // ...业务操作
    this.options.container && this._initDom()
  }

  _initDom() {
    // ... 业务操作
    ReactDOM.render(<App {...this.options} />, this.container) // app组件是React组件
  }
  
  unmount() {
    this.options.container && ReactDOM.unmountComponentAtNode(this.options.container)
  }
}

export default SDKName
```

webpack配置文件例：

```javascript

const options = {
  entry: {
    SDKName: './src/index.js'
  },
  output: {
    // ...其他配置
    filename: '[name].min.js',
    library: 'SDKName',
    libraryTarget: 'window',
    libraryExport: 'default',
  }
  // ... 其他常规配置
}

```
 
这样便能实现打包时生成一个SDKName.min.js的文件，并且该文件载入之后会将SDKName这个class类挂载到window对象上，以供调用

使用例：
```html
<script src="https://someWebSite.com/SDKName.min.js"></script>
<div id="root"></div>
<script>
  const sdk = new SDKName({
    container: document.querySelector('#root')
    // ...其他参数
  })
</script>
```

