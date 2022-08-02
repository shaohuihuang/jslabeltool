<template>
  <div>
    <label>
      <b>说明：1. 左键画点，右键闭合多边形(点数>2); 2. 左键可以点选多边形，每次只能选中一个; 3. 画完一个多边形后，自动切换到选取图元状态; 4. 支持滚轮缩放
      <br />
      注意："加载选区"必须在所有绘图操作之前执行，并且只能执行一次，否则会导致绘图操作的undo和redo混乱；undo和redo仅针对"绘制多边形"操作，删除操作无法撤销。
      </b>
    </label><br />
    辅助函数：<button id="setBackgroundImage" title="更换背景图片" @click="setBackgroundImage('tulips.jpg')">更换背景图片</button>
    <button id="loadPolygons" title="加载选区" @click='loadPolygons([ [ { "x": 172, "y": 261 }, { "x": 148, "y": 264 }, { "x": 138, "y": 287 }, { "x": 153, "y": 309 }, { "x": 155, "y": 341 }, { "x": 165, "y": 369 }, { "x": 199, "y": 397 }, { "x": 222, "y": 429 }, { "x": 298, "y": 437 }, { "x": 333, "y": 401 }, { "x": 349, "y": 358 }, { "x": 359, "y": 289 }, { "x": 338, "y": 267 }, { "x": 322, "y": 243 }, { "x": 317, "y": 221 }, { "x": 302, "y": 215 }, { "x": 273, "y": 213 }, { "x": 248, "y": 225 }, { "x": 228, "y": 245 }, { "x": 220, "y": 257 }, { "x": 201, "y": 258 }, { "x": 188, "y": 264 }, { "x": 172, "y": 261 } ], [ { "x": 337, "y": 167 }, { "x": 399, "y": 149 }, { "x": 442, "y": 176 }, { "x": 457, "y": 206 }, { "x": 471, "y": 248 }, { "x": 478, "y": 300 }, { "x": 494, "y": 328 }, { "x": 510, "y": 347 }, { "x": 509, "y": 362 }, { "x": 499, "y": 371 }, { "x": 461, "y": 368 }, { "x": 398, "y": 341 }, { "x": 373, "y": 308 }, { "x": 363, "y": 293 }, { "x": 348, "y": 243 }, { "x": 319, "y": 212 }, { "x": 320, "y": 179 }, { "x": 337, "y": 167 } ] ])'>加载选区</button>
    <button id="zoomIn" title="放大" @click="zoomIn(1.2)">放大1.2倍</button>
    <button id="zoomOut" title="缩小" @click="zoomOut(1.2)">缩小1.2倍</button>
    <button id="zoomReset" title="缩小" @click="zoomReset()">原始大小</button>
    <div id="coordinate">当前多边形坐标（终点=起点）：{{ coordinate }}</div>
    <br />
    <button id="selectObj" title="选取图元" @click="selectObj">选取图元</button>
    <button id="deleteObj" title="删除选中图元" @click="deleteObj">删除选中图元</button>
    <button id="drawPolygon" title="绘制多边形" @click="drawPolygon">绘制多边形</button>
    <button id="undo" title="后退" @click="undo()">后退</button>
    <button id="redo" title="前进" @click="redo()">前进</button>
    <br /><br />
    <canvas id="canvas" class="canvas" width="512" height="512"></canvas>
  </div>
</template>
<script>
import { fabric } from "fabric";
export default {
  data() {
    return {
      canvas: null,
      id: 0, //多边形编号，从0开始自动递增
      history: [], //历史记录，形如{"x":1,"y":1,"button":1,"id":0}
      currIndex: -1, //最新操作的history下标，undo和redo会用到 
      lines: [], //当前未闭合多边形的Line对象数组
      operation: "", //当前操作名称，目前只有polygon，为空表示选择
      coordinate: [] //坐标数组，每个元素是个json数组，代表一个多边形
    };
  },
  computed: {},
  mounted() {
    this.$nextTick(() => {
        this.canvas = new fabric.Canvas("canvas", {
          fireRightClick: true, // 启用右键, button=3
          stopContextMenu: true // 禁用右键菜单
        });
        this.fabricEvent();
    });
  },
  methods: {
    //选取图元，默认状态
    selectObj() {
      this.operation = "";
      this.canvas.selection = false; //禁止框选对象
      this.canvas.defaultCursor = 'default'; //默认光标
      this.canvas.getObjects().forEach((item) => {
        item.evented = true;
      });
      this.canvas.requestRenderAll();
    },
    //删除选中图元
    deleteObj() {
      this.operation = "";
      this.canvas.defaultCursor = 'default'; //默认光标
      this.canvas.getActiveObjects().forEach((item) => {
        this.canvas.remove(item);
      });
      //重算坐标
      this.coordinate = [];
      this.canvas.getObjects().forEach((item) => {
        //可能有未完成的多边形（线段），这里需要判断
        if (item.type == "polygon") {
          this.coordinate.push(item.points);
        } 
      });
      //取消选择
      this.canvas.discardActiveObject();
      this.canvas.requestRenderAll();
    },
    //绘制多边形
    drawPolygon() {
      this.operation = "polygon";
      //this.canvas.selection = false;
      this.canvas.defaultCursor = 'crosshair'; //默认光标改成十字
      this.canvas.getObjects().forEach((item) => {
        item.evented = false;
      });
    },
    //后退
    undo() {
      if (this.currIndex < 0)
        return;

      let op = this.history[this.currIndex];
      if (op.button == 1) {
        //回退左键操作
        this.currIndex--;
        //删掉最后一根线
        let obj = this.lines.pop();
        this.canvas.remove(obj);
        this.canvas.requestRenderAll();

        //切换到绘图状态
        this.drawPolygon();
      }
      else if (op.button == 3) {
        //回退右键操作
        this.currIndex--;

        //删掉最后一个多边形
        let obj = null;
        this.canvas.getObjects().forEach((item) => {
          if (item.type == "polygon") {
            obj = item;
          }
        });
        if (obj != null) {
          this.canvas.remove(obj);
        }
        //删掉最后一个多边形坐标
        this.coordinate.pop();

        //修改多边形id
        this.id = op.id;

        //重新构建lines
        this.lines = [];
        let x1 = -1;
        let y1 = -1;
        this.history.forEach((v) => {
          if (v.button == 1 && v.id == this.id) {
            //第一个点不用画线
            if (x1 >= 0 && y1 >= 0) {              
              this.addLine([x1,y1,v.x,v.y]);              
            }
            x1 = v.x;
            y1 = v.y;            
          }
        });

        //补最后一条线(终点)
        this.addLine([x1,y1,x1,y1]);

        this.canvas.requestRenderAll();

        //切换到绘图状态
        this.drawPolygon();
      }
    },
    //前进
    redo() {
      if (this.currIndex >= this.history.length - 1)
        return;

      //说明前面撤销到初始状态了，这里单独处理
      if (this.currIndex < 0) {
        //从头开始
        this.currIndex = 0;
        let op = this.history[this.currIndex];

        //补最后一条线(终点)
        this.addLine([op.x,op.y,op.x,op.y]);

        this.canvas.requestRenderAll();

        //切换到绘图状态
        this.drawPolygon();        
        return;
      }

      //当前位置
      let x1 = this.history[this.currIndex].x;
      let y1 = this.history[this.currIndex].y;
      let id1 = this.history[this.currIndex].id;

      let op = this.history[this.currIndex + 1];
      if (op.button == 1) {
        //前进左键操作
        this.currIndex++;

        //删掉最后一根线
        let obj = this.lines.pop();
        this.canvas.remove(obj);

        //先补一条线，如果是新多边形的第一点则忽略
        if (id1 == op.id) {
          this.addLine([x1,y1,op.x,op.y]);
        }
        
        //补最后一条线(终点)
        this.addLine([op.x,op.y,op.x,op.y]);

        this.canvas.requestRenderAll();

        //切换到绘图状态
        this.drawPolygon();
      }
      else if (op.button == 3) {
        //前进右键操作
        this.currIndex++;

        //修改多边形id
        this.id = op.id;

        //关闭多边形
        this.closePolygon();

        this.canvas.requestRenderAll();
      }
    },
    //更换背景图片
    setBackgroundImage(imgPath) {
      this.canvas.setBackgroundImage(
        imgPath,
        this.canvas.renderAll.bind(this.canvas)
      )
    },
    //加载选区
    loadPolygons(polygons) {
      polygons.forEach((points) => {
        this.addPolygon(points);
      });
      this.canvas.requestRenderAll();
    },
    //放大
    zoomIn(scale) {
      let zoom = this.canvas.getZoom();
      zoom *= scale;
      this.canvas.setZoom(zoom);
      if (zoom >= 1) {
        this.canvas.setWidth(512*zoom);
        this.canvas.setHeight(512*zoom);
      }
    },
    //缩小
    zoomOut(scale) {
      let zoom = this.canvas.getZoom();
      zoom /= scale;
      this.canvas.setZoom(zoom);
      if (zoom >= 1) {
        this.canvas.setWidth(512*zoom);
        this.canvas.setHeight(512*zoom);
      }
    },
    //原始大小
    zoomReset() {      
      this.canvas.setZoom(1.0);
      this.canvas.setWidth(512);
      this.canvas.setHeight(512);
    },
    //以下是绘图相关的函数
    fabricEvent() {
      this.canvas.on({
        "mouse:down": e => {
          let x = Math.round(e.pointer.x / this.canvas.getZoom());
          let y = Math.round(e.pointer.y / this.canvas.getZoom());
            
          //记录鼠标历史操作
          if (this.operation == "polygon") {
            this.currIndex++;
            //清理历史记录，如果当前位置后面还有元素，先清空
            if (this.currIndex < this.history.length) {
              this.history.splice(this.currIndex, this.history.length - this.currIndex);
            }
            //记录最新操作
            this.history.push({"x":x, "y":y, "button":e.button, "id":this.id});            
          }

          //左键处理
          if (e.button == 1) {
            //如果正在绘制多边形
            if (this.operation == "polygon") {
              this.addLine([x,y,x,y]);
              this.canvas.requestRenderAll();
            }
          }
          else if (e.button == 3) { //右键处理
            //如果正在绘制多边形，且点数大于2的时候，结束绘制
            if (this.operation == "polygon" && this.lines.length > 2) {
              this.closePolygon();
            }
          }
        },
        "mouse:wheel":  e => {
          var delta = e.e.deltaY;
          var zoom = this.canvas.getZoom();
          zoom *= 0.999 ** delta;
          if (zoom > 3) zoom = 3;
          if (zoom < 0.5) zoom = 0.5;
          this.canvas.setZoom(zoom);
          if (zoom >= 1) {
            this.canvas.setWidth(512*zoom);
            this.canvas.setHeight(512*zoom);
          }
          e.e.preventDefault();
          e.e.stopPropagation();
        },
        "mouse:move": e => {
          let x = Math.round(e.pointer.x / this.canvas.getZoom());
          let y = Math.round(e.pointer.y / this.canvas.getZoom());
          if (this.lines.length > 0 && this.operation == "polygon") {
            this.lines[this.lines.length - 1].set({x2:x, y2:y});
            this.canvas.requestRenderAll();
          }
        }
      });
    },
    //闭合多边形
    closePolygon() {
      //console.log(this.history);
      //移除旧线，清空数组
      this.lines.forEach((item) => {
        this.canvas.remove(item);
      });
      this.lines = [];

      //从历史记录读取点坐标
      let points = [];
      this.history.forEach((v) => {
        if (v.button == 1 && v.id == this.id) //忽略右键操作
          points.push({"x":v.x, "y":v.y});
      });
      //再加一次起点
      points.push(points[0]);
      //id自增
      this.id++;

      this.addPolygon(points);

      this.canvas.requestRenderAll();
      
      //画完一个就退出绘图状态
      this.selectObj();
    },
    //新增一条线
    addLine(points) {
      let line = new fabric.Line(points, {
        strokeWidth: 1,
        selectable: false,
        stroke: "red"
      });
      this.lines.push(line);
      this.canvas.add(line);
    },
    //新增一个多边形
    addPolygon(points) {
      let left = points[0].x;
      let top = points[0].y;
      points.forEach((v) => {
        if (v.x < left) left = v.x;
        if (v.y < top) top = v.y;
      })

      let poly = new fabric.Polygon(points, {
        fill: "rgba(255,0,0,0.1)",
        strokeWidth: 1,
        stroke: "red",
        left: left,
        top: top,
        cornerStyle: "circle",
        cornerColor: "rgba(0,0,255,1)",
        centeredRotation: false,
        centeredScaling: false,
        hasControls: false,
        lockMovementX: true,
        lockMovementY: true
      });

      this.coordinate.push(points);
      this.canvas.add(poly);
    }
  }
};
</script>
<style scoped>
.canvas {
  border: 1px solid black;
}
</style>