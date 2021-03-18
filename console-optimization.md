# 关于B端项目开发中的若干问题的处理方式

## 适用范围
- React
- Antd
- 后台项目

##常见问题

### 1. 表单提交

实践中常见的表单数据交互流程：

 ![流程图](https://res-qn.baoshiyun.com/91351181301202esbt.png)

**问题**

- 高耦合
> 输入复杂数据结构字段的控件实际上不承担数据处理功能，直接提交到Form后交由业务组件触发Form的OnChange处理，处理结束后存储到redux等处，造成表单控件与业务组件的耦合

- 低复用
> 如果复杂数据结构字段控件不承担数据处理功能的话，若业务上需要复用该控件，就必须连处理数据和存储数据的业务逻辑一并复制，甚至可能包括redux的相关内容，在业务逻辑复杂时往往及其困难

- 难定位问题
> 进行处理数据的步骤过多，出现数据问题时需要逐步回溯，找出哪一步真正出了问题才能进行处理

---

理想的表单数据交互：

![流程图](https://res-qn.baoshiyun.com/91351181301202mbvo.png)

**解决问题**

- 低耦合
> 对于复杂输入组件，结构决定性质，绝大多数时候也是输入方式和展示方式决定传出的数据格式（如时间选择控件、图片上传控件、列表内容选择控件等）。

> 因此对于同一种展示方式的组件，将 **服务器数据->展示用的数据** 的变化过程封装在组件内部，则数据的受影响范围也就限制在组件内部，强制无法与props以外的数据产生交互，降低耦合性

- 高复用
> 业务上要复用输入组件时，如果是同个项目里的复用，直接引用即可，不同项目里的复用也可以通过简单复制该组件相关的内容进行，边界比较清晰，复用比较简单

- 容易定位问题
> 因为只有组件内部进行数据处理，如果提交时发现问题，只需要关注组件内部的数据流转过程即可，不需要在业务组件、form表单和输入控件之间反复横跳寻找问题

- **容易提取为公用组件**
> 进行组件逻辑提取后即可成为该种输入类型下的公用组件

---

### *如何实现？*

- #### 理解ant-design Form.Item控制控件的本质

![img.png](https://res-qn.baoshiyun.com/61551181301202cldo.png)

> ant-design的Form.Item通过给包装的控件传入value和onChange（或类似的其他属性）来接管了该控件的数据传入（value）和传出（onChange），从而实现了表单和输入控件的双向绑定。
> 而用户通常会使用的输入控件就是ant-design提供的输入控件，它们都对ant-design的Form.Item已经完整配合。<br/>所以我们只需要用value和onChange作为props即可自己实现一个受Form.Item控制的高阶输入控件

通用示例：
```javascript
const CustomInput = ({value, onChange, ...restProps}) => { // 建立一个props中有value和onChange的函数组件
 
    const [innerValue, setInnerValue] = useState() // 在内部设立一个展示需要的数据的state
  
    const normalize = (val) => ({value: val}) // 序列化函数，form字段值->内部使用值
    const renormalize = (val) => val.value // 反向序列化函数，内部onChange获得的值->form字段值
    
    const innerOnchange = (val) => {
      setInnerValue(val)
      onChange(renormalize(val))
    }
  
    useEffect(() => { // 当value变化时进行内部变形，将form传入的值改为内部组件展示需要的值
      if(renormalize(innerValue) !== value) { // 需要判断该值是否为上次内部组件变化时onChange传出的，防止无限循环赋值
        setInnerValue(normalize(value))
      }
    }, [value])

    // 可以是任意的组件，接管属性不一样，但属性的赋值和变化逻辑均被本组件接管即可
    // 其他参数可透传可选择传入
    return <SomeComponent onChange={innerOnchange} value={innerValue} {...restProps}/> 
}
```
应用该示例方法可以实现绝大多数复杂输入组件 <br>
包括但不限于：时间选择控件、图片上传控件、列表内容选择控件等

- #### 隐含的条件
    服务器要求我们上传的数据结构，应当与回显时从API返回的一致，需要努力与后端开发 ***沟（zheng）通（chao）***
  
<br><br><br><br>

### 2. 列表页的表格、搜索栏的控制权归属

我们尝试过的类型：

![结构图](https://res-qn.baoshiyun.com/91351181301202mzxz.png)
 <br/> 该类型下网络请求在搜索栏组件内发起️

![结构图](https://res-qn.baoshiyun.com/91351181301202mxhx.png)
 <br/> 该类型下网络请求在列表展示组件内发起

**问题**

无论网络请求处于搜索栏组件内还是列表展示组件内，当没有网络请求的那个组件需要从内部发起网络请求时都会造成超长的调用链 <br>
而且因为业务逻辑不清，最后会造成搜索栏组件和列表展示组件均不够独立，无法解耦

### ***如何解决？***

受React-beautiful-dnd启发，使用react的自定义hooks实现一个可以同时控制搜索栏组件和列表展示组件的自定义业务逻辑hook


结构如下：<br>
![结构图](https://res-qn.baoshiyun.com/91351181301202ciqj.png)
<br>
在该结构下，网络请求位于usePaginationList的hooks内，双方都可以通过hooks返回的方法进行搜索操作，同时搜索栏组件的Form实例使用的也是hooks内部生成的Form实例，当搜索栏的输入值有改变时可以同步给hooks，保证任何时候任何对象触发Form的submit都会用当前显示的搜索条件进行搜索

#### usePaginationList的代码
```javascript
export const SEARCH_TYPE = {
  REPLACE: 1,
  ADD: 2
}


/**
 * @description 用于分页列表的自定义hooks
 * @author yang xin
 * @param {object} params: 自定义传入参数，会在发送http请求时覆盖默认的从form中取得的fieldsValues
 * @param {string} url: 获取列表的接口地址 // todo 默认所有查询接口都应该是使用get方式，如果以后有以post方式请求的查询接口请重载该hook
 * @param {object} headers?: 自定义请求头
 * @param {number} pageSize?: 自定义 pagesize
 * @param {object} customRequest?: 自定义请求方法
 * @param {number} type?: 获取列表之后的处理方式类型，默认为覆盖当前列表，在C端等无限下拉列表中可能是add方式，即添加在当前列表之后
 * @return {[searchFormProps, tableProps]} 返回一个数组，第一个元素是给searchForm用的props，第二个是给table用的props，解构赋值给相应的组件
 */
export const usePaginationList = ({
  params,
  url,
  customRequest,
  pageSize: ps,
  headers = {},
  type = SEARCH_TYPE.REPLACE
}) => {
  const [init, setInit] = useState(false) // 初始化时防止自动搜索

  const [list, setList] = useState([])

  const [page, setPage] = useState(1)

  const [pagesize, setPageSize] = useState(ps || 20)

  const [total, setTotal] = useState(0)

  const [isLoading, setLoading] = useState(false)

  const [form] = Form.useForm()

  /**
   * @description 获取列表的请求
   * @author yang xin
   * @param {{[p: string]: any; pagesize: number; page: number}} reqParams: 请求参数
   * @return {type} description
   */
  const requestPaginationList = async (reqParams = {}) => {
    setLoading(true)

    const formValues = form.getFieldsValue()

    try {
      let res
      if (customRequest) {
        res = await customRequest(reqParams)
      } else {
        res = await request.get(url, { ...formValues, ...reqParams })
      }
      // const res = await request.get(url, { ...formValues, ...reqParams })
      setLoading(false)
      if (res) {
        let newList = list
        if (type === SEARCH_TYPE.REPLACE) {
          newList = res.data.list || [] // 根据实际接口返回替换
        }
        if (type === SEARCH_TYPE.ADD) {
          newList = [...list, ...res.data.list] // 根据实际接口返回替换
        }
        // await timer(500).toPromise()
        console.log('newList::', newList)
        
        setList(newList)
        setPage(res.data.page) // 根据实际接口返回替换
        setTotal(res.data.total) // 根据实际接口返回替换
      }



    } catch (e) {
      console.error(e)
      setLoading(false)
    }

  }
  // 通过form查询时，把页码设置为 1
  const formSearch = (customParams = {}) => {
    if (page === 1) {
      return requestPaginationList({
        page,
        pagesize,
        ...customParams
      })
    }

    setPage(1)
  }

  const onSearch = (customParams = {}) => {
    if (!isLoading) {
      return requestPaginationList({
        page,
        pagesize,
        ...customParams
      })
    }
  }

  useEffect(() => {
    if (init) {
      let reqParams = {
        page,
        pagesize
      }
      if(params && !params.isInitParams) reqParams = {...reqParams, ...params}
      requestPaginationList(reqParams)
    } else {
      setInit(true)
    }
  }, [page, url, pagesize])

  // useEffect(() => {console.log('loading change', isLoading)}, [isLoading])

  return [
    { form, isLoading, onSearch: formSearch }, // 给searchFrom使用的props
    {
      dataSource: list,
      list, 
      isLoading, 
      total, 
      page, 
      current: page,
      pageSize: pagesize,
      onSearch,
      changePage: setPage, 
      setPage, 
      setTotal,
      setPageSize,
      pagination: {
        current: page,
        pageSize: pagesize,
        onChange: setPage,
        total,
        showSizeChanger: false
      }
    } // 给ant-design的table使用的props，dataSource和pagination属性完全符合antd的table要求，也提供了部分自定义方法和数据便于自己调取
  ]
}
```
使用例：
```javascript
    const [searchProps, tableProps] = usePaginationList({url: '/api/xxx'})

    const conditions = [...根据业务进行配置]
    const columns = [...根据业务进行配置]

    return(
      <div>
        <SearchForm  {...searchProps} conditions={conditions}/>
        <Table {...tableProps} columns={columns}/>
      </div>
    )
```

可以看到，使用了该hooks之后，关于列表的通用逻辑，如：pagination管理（翻页、换页、页码组件的选项等）和列表的搜索、重置等功能均被hooks代理
<br>业务组件可以专注创建业务相关的内容，如：searchForm的搜索配置项和table的行配置等
<br>而SearchForm和table也可以从内部直接调用搜索、换页等列表相关的功能  

- #### 隐含的条件
    1. 需要SearchForm的Form中的值只要取出之后就可以直接交给搜索http请求，中间不应该进行变形，所以需要先实现一个符合我们上面的写法的SearchForm组件，才能实现
    2. 一个项目中的分页搜索请求的分页相关参数的key必须一样，如page和pageSize，如果一个是page，一个是current，或者一个是pagesize，一个是pageSize，请 **和后端努力吵架**，也可以考虑给hook加入新的参数传入相关的key
    


### 3. 一些小Tips

#### 1.使用枚举对象
http请求中传入或返回的表示类型的特定字段值，建议以枚举对象的方式保存在consts文件夹下
例：
```jsx
const SOME_TYPES = {
  TYPE_ONE: 1,
  TYPE_TWO: 2,
}

const SOME_TYPE_NAMES = {
  [SOME_TYPES.TYPE_ONE]: '类型1',
  [SOME_TYPES.TYPE_TWO]: '类型2'
}

// 在输入组件中使用
const selectTypes = Object.keys(SOME_TYPE_NAMES)

return <Radio.Group>
  {
    selectTypes.map( key => <Radio value={key}>{SOME_TYPE_NAMES[key]}</Radio>)
  }
 </Radio.Group>

// 在请求的返回中使用
const res = await request()

if(res.someType === SOME_TYPES.TYPE_ONE) {
  // 业务逻辑
}
if(res.someType === SOME_TYPES.TYPE_TWO) {
  // 业务逻辑
}
// 或者
const processes = {
  [SOME_TYPES.TYPE_ONE]: (props) => {
    // 业务逻辑
  },
  [SOME_TYPES.TYPE_TWO]: (props) => {
    //业务逻辑
  }
}

const selectedProcess = processes[res.someType]
selectedProcess && selectedProcess(res)
// 好处是省去了ifelse判断，更多type的情况下也可以省去switch判断

```


#### 2.表单中使用Form.Item作为样式容器和逻辑容器

用作样式容器：<br> 因为antd的Form.Item传入name时只允许有且仅有唯一的children元素，所以当设计给出输入控件的复杂设计时，如果逻辑不复杂，可以考虑使用一个外层的Form.Item作为样式容器
```jsx
// 外层formItem只作为样式容器，不传入name参数
return <Form.Item style={...styles} label='someLabel' required> 
 {/* 内层FormItem作为实际的表单控制项 */}
  <Form.Item name='someName' noStyle>
    <Input />
  </Form.Item>
  <div>
    {/*设计给出的复杂设计中添加的其他元素*/}
  </div>
</Form.Item>
```

用作逻辑容器：<br> 常见于依赖其他表单项控制展示与否的字段
```jsx

return <Form.Item shouldUpdate noStyle>
  {
    (form) => {
      if(form.getFieldValue('someKey') === 'someValue') {
        return null
      }
      return <Form.Item name='someOtherKey'>
        <Input />
      </Form.Item>
    }
  }
</Form.Item>
```

以逻辑容器使用方式展开，建议在开发过程中将不同类型的输入控件组单独作为组件分离，以实现更好的控制和逻辑分割
