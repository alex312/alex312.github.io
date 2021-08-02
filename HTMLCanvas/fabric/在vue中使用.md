# 在Vue中使用Fabric.js

## 安装

```cmd
npm install fabric --save
```

## 在组件中引用

```js
import { fabric } from 'fabric'
```
注意，引用时需要在使用‘{}’引用，否则会出现如下错误
```
Vue warn]: Error in mounted hook: "TypeError: fabric__WEBPACK_IMPORTED_MODULE_0___default.a.Canvas is not a constructor"
....
```

## 对象

### Image

- 通过下面的方法直接通过url加载图片
    ```js
    fabric.Image.fromURL(require('<图片在项目中的路径>'), (img)=>{
        // 对图片的设置和操作
        ...
        // 将图片添加到canvas对象中
        this.canvas.add(img)
    })
    ```
    需要注意的是注意引用图片的时，需要使用require。否则页面将会出现无法找到资源的错误

- 通过img标签加载图片
  ```vue
  <template>
    <div>  
        <img src="project/path/to/image" id="img"/>
        <canvas id="canvas"/>
    </div>
  </template>
  <script>
  import { fabric } from 'fabric'
  export default {
      ...
      mounted(){
        ...

        let imgE = document.getElementById('img')
        let img = fabric.Image(imgE,{})
        
        ...

        this.canvas.add(img)
      }
  }
  </script>
  ```

- 加载SVGElement
  ```vue
  // TODO:

  ```

- 

## 问题

### [未解决] 重设对象的left和top属性后，鼠标点击图形无法选中对象，对象的可选中区域在初始最初的left和top定义的位置
代码如下
```js
    this.canvas = new fabric.Canvas('station-map')

    const rect2 = new fabric.Rect({
      left: 100,
      top: 100,
      fill: 'red',
      width: 20,
      height: 20,
      angle: 45
    })
    this.canvas.add(rect2)

    rect2.set({ left: 20, top: 50 })
    this.canvas.renderAll()
```