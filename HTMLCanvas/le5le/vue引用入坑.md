
1. WEB拓扑图 vue-element-admin + le5le 
yarn add  topology-core topology-class-diagram topology-activity-diagram topology-flow-diagram topology-sequence-diagram -D 
安装过程中失败

2. 使用 topology vue 项目安装成功，也是坑多多

```vue
<template>
    <div>
      <div class="canvas" id="container"></div>   
    </div>
</template>
 
<script>
import * as Le5le from 'topology/core'
//import { Topology, Node, Line } from 'topology/core' 

export default {
  name: 'Le5leTest',
  data() {
    return {
      canvas: null,
      json: null
    }
  },
  methods: {
    init: function() {
        let container = document.getElementById('container');
        var width = container.clientWidth;
	      var height = container.clientHeight;
        this.canvas = new Le5le.Topology('container', {
            width,
            height,
            on: function(event, data) {
              //console.log(event, data);    
          }});
	
      // 空白数据图形数据，可以来自于官网下载的json
      const json = {nodes:[], lines:[]};
      //this.json = {"nodes":[{"id":"069a4fa7","name":"rectangle","tags":[],"rect":{"x":16,"y":26,"width":100,"height":100,"center":{"x":66,"y":76},"ex":316,"ey":626},"lineWidth":1,"rotate":0,"offsetRotate":0,"globalAlpha":1,"dash":0,"strokeStyle":"#222","fillStyle":"","font":{"color":"#222","fontFamily":"\"Hiragino Sans GB\", \"Microsoft YaHei\", \"Helvetica Neue\", Helvetica, Arial","fontSize":12,"lineHeight":1.5,"fontStyle":"normal","fontWeight":"normal","textAlign":"center","textBaseline":"middle"},"animateStart":0,"animateCycleIndex":0,"lineDashOffset":0,"text":"NODE1","textOffsetX":0,"textOffsetY":0,"animateType":"","data":"","zRotate":0,"anchors":[{"x":216,"y":576,"direction":4},{"x":266,"y":526,"direction":1},{"x":316,"y":576,"direction":2},{"x":266,"y":626,"direction":3}],"rotatedAnchors":[{"x":216,"y":576,"direction":4},{"x":266,"y":526,"direction":1},{"x":316,"y":576,"direction":2},{"x":266,"y":626,"direction":3}],"animateDuration":0,"animateFrames":[],"borderRadius":0,"icon":"","iconFamily":"topology","iconSize":null,"iconColor":"#2f54eb","imageAlign":"center","bkType":0,"gradientAngle":0,"gradientRadius":0.01,"paddingTop":10,"paddingBottom":10,"paddingLeft":10,"paddingRight":10,"paddingLeftNum":10,"paddingRightNum":10,"paddingTopNum":10,"paddingBottomNum":10,"textRect":{"x":226,"y":596,"width":80,"height":20,"center":{"x":266,"y":606},"ex":306,"ey":616},"fullTextRect":{"x":226,"y":536,"width":80,"height":80,"center":{"x":266,"y":576},"ex":306,"ey":616},"iconRect":{"x":226,"y":536,"width":80,"height":55,"center":{"x":266,"y":563.5},"ex":306,"ey":591},"fullIconRect":{"x":226,"y":536,"width":80,"height":80,"center":{"x":266,"y":576},"ex":306,"ey":616}}],"lines":[],"lineName":"polyline","fromArrowType":"","toArrowType":"triangleSolid","scale":1,"locked":0};
		
      this.canvas.open(this.json);
    },



    addNode: function ()
	  {
        var nodeStr = `
        {
          "text": "nodeName",
          "tags": ["testNum"],
          "rect": {
            "width": 50,
            "height": 70
          },
          "is3D": true,
          "z": 10,
          "zRotate": 15,
          "fillStyle": "#f00",
          "name": "rectangle",
          "icon": "\ue63c",
          "iconFamily": "topology",
          "iconColor": "#777",
          "iconSize": 50
        }` ;
      
      const node = JSON.parse(nodeStr);
      node.rect.x = 100; // event.offsetX - ((node.width / 2) << 0);
      node.rect.y = 100; //event.offsetY - ((node.height / 2) << 0);
      
      //this.json.nodes.push(node);
      //console.log('data:',this.json);
      //this.canvas.open(this.json);

      var topoNode = new Le5le.Node(node);
      this.canvas.addNode(topoNode, true);
      //console.log('data:',this.canvas.data.nodes);
    }
    
    /////////////////////////////////////
  },
  mounted() {
      this.init();
      this.addNode();
  }
}
</script>
<style scoped>
  #container {
    height: 400px;
  }
</style>
```

以上例子加载显示成功，但是操作会有问题，分析是VUE-E-A的侧边栏造成的X方向定位问题
全屏后坐标正确，再缩放也正确，但是侧边栏切换后坐标错误

勉强解决vue-element-admin + le5le侧边栏导致的鼠标操作偏移问题，需要通过element-resize-detector监听页面变化，并调整boundingRect大小
    this.le5leCanvas.boundingRect = container.getBoundingClientRect();
但是初始化时还是存在偏移，即使是在mounted的$nextTick中调整也不行，是页面加载的切入效果导致？只好用延时1秒解决（需要使用watch？没有找到更好办法，）

```js
resetTopoSize: function()
    {
        var container = document.getElementById('container');
        var rc = this.le5leCanvas.divLayer.canvas.getBoundingClientRect();
        var divRc = container.getBoundingClientRect();
        console.log("Le5le Bound box:" + rc.x + "," + rc.y + "," + rc.width + "," + rc.height);
        console.log("Canvas bound box:" + divRc.x + "," + divRc.y + "," + divRc.width + "," + divRc.height);

        //divRc.x = 0;
        this.le5leCanvas.boundingRect = divRc;
        this.le5leCanvas.resize();
        this.le5leCanvas.render();
    },

	
	 mounted() {

      var self = this;
      self.init();

      var elementResizeDetectorMaker = require("element-resize-detector");//导入
      var erd = elementResizeDetectorMaker();

      //监听id为test的元素 大小变化
      erd.listenTo(document.getElementById("container"), function(element) {
        var width = element.offsetWidth;
        var height = element.offsetHeight;
        //console.log("Size: " + width + "x" + height);

        //var myEvent = new Event('resize');
        //window.dispatchEvent(myEvent);
        if(self.le5leCanvas != null)
        {
          self.resetTopoSize();
        }
      });
      
      this.$nextTick(() => {
        console.log("Next tick begin");
        //document.getElementById("addBtn").click();
        setTimeout(() => {
          self.resetTopoSize();
        }, 1000);
        console.log("Next tick end");
      });
  }
  ```