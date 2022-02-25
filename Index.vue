<template>
  <div class="five-main">
    <div id="myDiagramDiv" ref="elRef" class="gojs"></div>
    <div class="gojs-tool-bar">
      <el-tooltip class="item" effect="dark" content="自适应" placement="left">
        <i class="el-icon-aim" @click="zoomToFit()"></i>
      </el-tooltip>
      <div class="gojs-overview-container" id="myOverviewDiv"></div>
    </div>
  </div>
</template>

<script>
import go from 'gojs'
import DoubleTreeLayout from './DoubleTree'
export default {
  data() {
    return {
      images: require('@/assets/image/01.png'),
      myDiagram: null,
      timer: null,
      linkDataArray: [
        { from: 1, to: 2 },
        { from: 1, to: 3 },
        { from: 3, to: 4 },
        { from: 4, to: 5 },
        { from: 4, to: 6 },
        { from: 3, to: 7 },
        { from: 3, to: 8 },
        { from: 1, to: 'Left1' }
      ],
      nodearray: [
        { key: 1, name: "开始", color: '#2a6dc0', category: "Root" },
        { key: 2, name: "不符合", color: '#ea2857' },
        { key: 3, name: "符合预期", color: '#2a6dc0' },
        { key: 4, name: "结算", color: '#2a6dc0' },
        { key: 5, name: "结算一", color: '#1cc1bc' },
        { key: 6, name: "计算二", color: '#2a6dc0' },
        { key: 7, name: "校验", color: '#ea2857' },
        { key: 8, name: "审核", color: '#ea2857' },
        { key: "Left1", name: '左侧的', dir: "left", color: 'red', category: "LeftNode" },
      ],
      HORIZONTAL: true,
      goMake: null,
      BandedTreeLayout: null,
      activeMindmapNodeData: null,// 当前选中的节点数据
    }
  },
  methods: {
    initGojs(props, context) {
      function BandedTreeLayout() {
        go.TreeLayout.call(this);
        this.layerStyle = go.TreeLayout.LayerUniform;
      }
      go.Diagram.inherit(BandedTreeLayout, go.TreeLayout);

      BandedTreeLayout.prototype.commitLayers = function (layerRects, offset) {
        // update the background object holding the visual "bands"
        var bands = this.diagram.findPartForKey("BANDS");
        if (bands) {
          var model = this.diagram.model;
          bands.location = this.arrangementOrigin.copy().add(offset);

          // make each band visible or not, depending on whether there is a layer for it
          for (var it = bands.elements; it.next();) {
            var idx = it.key;
            var elt = it.value;  // the item panel representing a band
            elt.visible = idx < layerRects.length;
          }

          // set the bounds of each band via data binding of the "bounds" property
          var arr = bands.data.itemArray;

          for (var i = 0; i < layerRects.length; i++) {
            var itemdata = arr[i];
            if (itemdata) {
              model.setDataProperty(itemdata, "bounds", layerRects[i]);
            }
          }
        }
      };
      this.BandedTreeLayout = BandedTreeLayout
      this.goMake = go.GraphObject.make;
      this.init()
    },
    init() {
      // if (window.goSamples) goSamples();  // init for these samples -- you don't need to call this
      this.myDiagram = this.goMake(go.Diagram, this.$refs.elRef,
        {
          layout: this.goMake(DoubleTreeLayout,  // BandedTreeLayout  DoubleTreeLayout TreeLayout
            {
              directionFunction: n => n.data && n.data.dir !== "left",
              /* angle: this.HORIZONTAL ? 0 : 90,
              arrangement: this.HORIZONTAL ? go.TreeLayout.ArrangementVertical : go.TreeLayout.ArrangementHorizontal, */
            }),

          isReadOnly: false, // 只读
          "undoManager.isEnabled": true,
          initialContentAlignment: go.Spot.Center,
          initialAutoScale: go.Diagram.Uniform,
          "toolManager.mouseWheelBehavior": go.ToolManager.WheelZoom,
          "commandHandler.deletesTree": true,
        });

      this.myDiagram.nodeTemplate = this.getNodeTemplate('right')
      this.myDiagram.nodeTemplateMap.add("LeftNode", this.getNodeTemplate('left'));
      this.myDiagram.nodeTemplateMap.add("Root", this.getNodeTemplate('root'));
      this.myDiagram.linkTemplate =
        this.goMake(go.Link,//MultiArrowLink Link
          { curviness: 2, corner: 5 },
          go.Link.Orthogonal,//连线样式 go.Link.Bezier  Orthogonal go.Link.Orthogonal,
          //连接线颜色设置
          this.goMake(go.Shape, { strokeWidth: 1, stroke: '#1296db' }, new go.Binding("stroke", "color")),
          // 箭头样式设置
          this.goMake(go.Shape, { toArrow: "Standard", scale: 0.5, strokeWidth: 1, stroke: '#1296db', fill: '#1296db' }),

        );
      //右键事件
      //   this.addContextMenu()
      this.myDiagram.model = this.goMake(go.GraphLinksModel,
        {
          copiesArrays: true,
          copiesArrayObjects: true,
          nodeDataArray: this.nodearray,
          linkDataArray: this.linkDataArray
        });
      // this.loop()
      // 初始化鹰眼
      this.initOVerview()
    },
    /**
     * 鹰眼、缩略图
     */
    initOVerview() {
      this.goMake(go.Overview, "myOverviewDiv",
        {
          observed: this.myDiagram,
          contentAlignment: go.Spot.Center
        });
    },
    /**
    * 节点模板
     * @param {*}  type 模板类型
     */
    getNodeTemplate(type) {
      return this.goMake(go.Node, go.Panel.Horizontal,
        {
          click: (a, b,) => {
            //  console.log(b.data)
          },
          selectionChanged: (part) => {
            // 保存当前选中的节点数据
            this.activeMindmapNodeData = part.data
            this.$emit('selected', {
              isSelected: part.isSelected,
              nodeData: part.data
            })
          },
          pickable: true,
          selectable: true,
          resizable: false,
          selectionAdorned: false,
          textEditable: true,
          fromLinkable: true,
          toLinkable: true,
          portId: "",
        },
        new go.Binding("visible"),
        // 折叠按钮
        ['left', 'root'].includes(type) ? this.getCollapseButton(type, 'left') : '',
        this.goMake(go.Panel, go.Panel.Auto,
          {
            // margin: new go.Margin(0, 0, 0, 0),
            // minSize: new go.Size(120, 40),
            fromLinkable: false,
            toLinkable: false,
          },
          this.goMake(go.Shape, "RoundedRectangle",
            {
              name: "SHAPE",
              minSize: new go.Size(120, 40),
              stroke: null,
              cursor: "pointer",
              fromLinkable: false,
              toLinkable: false,
            },
            new go.Binding("minSize", "size", function (b) {
              return b ? new go.Size(b.w, b.h) : new go.Size(120, 40);
            }),
            // 背景色
            new go.Binding("fill", "isSelected", function (b, item) {
              return b ? '#19db1e' : item.part.data.color || '#397bf7';
            }).ofObject(),
          ),
          this.goMake(go.TextBlock, this.textStyle(),
            {
              name: "NAMETB",
              margin: 0,
              //   row: 0, column: 0, columnSpan: 5,
              font: "12pt Segoe UI,sans-serif",
              editable: true,
              isMultiline: false,
              // minSize: new go.Size(10, 16)
            },
            new go.Binding("text", "name").makeTwoWay()
          ),
        ),
        // 折叠按钮
        ['right', 'root'].includes(type) ? this.getCollapseButton(type, 'right') : '',
        { // this tooltip Adornment is shared by all nodes
          toolTip:
            this.goMake("ToolTip",
              this.goMake(go.TextBlock, { margin: 4 },  // the tooltip shows the result of calling nodeInfo(data)
                new go.Binding("text", "name"))
            ),
          // this context menu Adornment is shared by all nodes
          contextMenu: this.addContextMenu()
        },
      );
    },
    getCollapseButton(type, dir) {
      const visibleKey = type === 'root' ? (dir === 'left' ? 'lVisible' : 'rVisible') : "avisible";
      return this.goMake("Button",
        {
          visible: false,
          // alignment: go.Spot.Right,
        },
        new go.Binding('visible', "isTreeLeaf",
          function (leaf) { return !leaf; })
          .ofObject(),
        this.goMake(go.Shape,
          {
            name: "ButtonIcon",
            figure: "MinusLine",
            desiredSize: new go.Size(6, 6)
          },
          new go.Binding("figure", visibleKey,//"isCollapsed",
            function (collapsed) {
              return collapsed ? "PlusLine" : "MinusLine";
            })),
        {
          click: (e, obj) => {
            e.diagram.startTransaction();
            var node = obj.part;
            if (node.data[visibleKey]) {
              this.expandFrom(node, node, type, dir, visibleKey);
            } else {
              this.collapseFrom(node, node, type, dir, visibleKey);
            }
            e.diagram.commitTransaction("toggled visibility of dependencies");
          }
        })

    },
    // 文字样式
    textStyle() {
      return {
        stroke: "#fff",
        name: 'boss',
        font: "18px sans-serif",
        textAlign: "center",
        wrap: go.TextBlock.WrapBreakAll,// TextBlock.WrapDesiredSize, TextBlock.
        verticalAlignment: go.Spot.Center,//文字垂直居中
        overflow: go.TextBlock.OverflowEllipsis,
        editable: true,
        margin: new go.Margin(0, 0, 0, 0),
      }
    },

    setLinks(model, fromData, toData) {
      let linkdata = {
        from: model.getKeyForNodeData(fromData),
        to: model.getKeyForNodeData(toData)
      };
      // 添加连线
      model.addLinkData(linkdata);
    },
    //新增节点重叠部分处理
    shiftNodesToEmptySpaces() {
      this.myDiagram.selection.each((node) => {
        if (!(node instanceof go.Node)) return;
        // look for Parts overlapping the node
        while (true) {
          let exist = this.myDiagram.findObjectsIn(node.actualBounds,
            // only consider Parts
            function (obj) { return obj.part; },
            // ignore Links and the dropped node itself
            function (part) { return part instanceof go.Node && part !== node; },
            // check for any overlap, not complete containment
            true).first();
          if (exist === null) break;
          // try shifting down beyond the existing node to see if there's empty space
          node.position = new go.Point(node.actualBounds.x, exist.actualBounds.bottom + 10);
        }
      });
    },
    //添加右键事件
    addContextMenu() {
      return this.goMake("ContextMenu",
        this.goMake("ContextMenuButton",
          this.goMake(go.TextBlock, " 添 加 ", {
            margin: 5,
            width: 60,
            textAlign: "center",
          }),
          {
            click: (e, obj) => {
              // this.visible = true;

              this.addNodeAndLink(e, obj, 'event');
            }
          },
          new go.Binding("visible", "", function (o) { return o.diagram && o.diagram.commandHandler.canDeleteSelection(); }).ofObject()
        ),
        this.goMake("ContextMenuButton",
          this.goMake(go.TextBlock, "编 辑", {
            margin: 5,
            width: 60,
            textAlign: "center",
          }),
          {
            click: (e, obj) => {
              // this.visible = true;
              this.editNode(e, obj, 'event');
            }
          },
          new go.Binding("visible", "", function (o) { return o.diagram && o.diagram.commandHandler.canDeleteSelection(); }).ofObject()
        ),


        this.goMake("ContextMenuButton",
          this.goMake(go.TextBlock, " 删 除 ", {
            margin: 5,
            width: 60,
            textAlign: "center",
          }),
          {
            click: (e, obj) => {
              const isRoot = obj.part.adornedPart.findTreeLevel()
              if (isRoot === 0) {
                this.$message.error('根节点禁止删除')
                return
              }
              this.$confirm('确定要删除节点吗')
                .then(_ => {
                  e.diagram.commandHandler.deleteSelection();
                  done();
                })
                .catch(_ => { });

            }
          },
          new go.Binding("visible", "", function (o) { return o.diagram && o.diagram.commandHandler.canDeleteSelection(); }).ofObject())
      );

    },
    /**
     * 折叠节点
     * @param {*} node 
     * @param {*} start
     * @param {*} type 点击的节点类型 root left right
     * @param {*} dir   节点按钮方向 left right
     */
    collapseFrom(node, start, type, dir, visibleKey) {
      if (node.data.isCollapsed) return;
      const isRoot = node.findTreeLevel() === 0
      visibleKey = isRoot ? visibleKey : 'avisible'
      // 根节点
      if (!node.data.dir && !isRoot) node.data.dir = 'right'
      // 折叠对应方向节点
      if (node.data.dir !== dir && !isRoot) return;
      if (node !== start) node.diagram.model.setDataProperty(node.data, 'visible', false);

      node === start && node.diagram.model.setDataProperty(node.data, visibleKey, true);
      node.findNodesOutOf().each((nodes) => {
        this.collapseFrom(nodes, null, type, dir, visibleKey)
      });
    },

    /**
     * 展开节点
      * @param {*} node 
     * @param {*} start
     * @param {*} type 点击的节点类型 root left right
     * @param {*} dir   节点按钮方向 left right
     */
    expandFrom(node, start, type, dir, visibleKey) {
      const isRoot = node.findTreeLevel() === 0
      visibleKey = isRoot ? visibleKey : 'avisible'
      // 根节点
      if (!node.data.dir && !isRoot) node.data.dir = 'right'
      // 展开对应方向节点
      if (!isRoot && node.data.dir !== dir) return;

      if (node !== start) node.diagram.model.setDataProperty(node.data, 'visible', true);

      node === start && node.diagram.model.setDataProperty(node.data, visibleKey, false);
      // 折叠节点自己可用状态不设置
      if (node.data[visibleKey]) return
      node.findNodesOutOf().each(nodes => {
        this.expandFrom(nodes, null, type, dir, visibleKey)
      });
    },


    /** 
     * 更新数据
    */
    loop() {
      let model = this.myDiagram.model;
      let arr = model.nodeDataArray;
      /* for (let i = 0; i < arr.length; i++) {
        let data = arr[i];
        if (!Object.prototype.hasOwnProperty.call(data, "signalStrength")) continue
        // 模拟信号强度
        let signalStrength = Math.ceil(Math.random() * 17)

        data.signalStrength = data.signalStrength.map(v => {
          v.color = v.val < signalStrength ? "red" : "#ececec"
          return v
        })
        model.updateTargetBindings(data);
      } */
      //连线颜色
      /*  arr = model.linkDataArray;
       for (let i = 0; i < arr.length; i++) {
         let data = arr[i];
         data.color = '#' + Math.floor(Math.random() * 16777215).toString(16);
         model.updateTargetBindings(data);
       } */
      this.myDiagram.links.each(function (link) {
        var shape = link.findObject("PIPE");

        var off = shape.strokeDashOffset - 3;

        // 设置（移动）笔划划动画

        shape.strokeDashOffset = (off <= 0) ? 60 : off;

        // 动画（频闪）不透明度：

        if (down) opacity = opacity - 0.01;

        else opacity = opacity + 0.003;

        if (opacity <= 0) { down = !down; opacity = 0; }

        if (opacity > 1) { down = !down; opacity = 1; }

        shape.opacity = opacity;

      });

      this.timer = setTimeout(() => {
        this.loop();
      }, 1000);
    },
    //添加一个子节点
    addNodeAndLink(e, obj, type) {
      //先添加泳道
      let fromNode = obj.part;
      let fromData = fromNode.data;

      this.$emit('addNode', {
        isTreeRoot: obj.part.adornedPart.findTreeLevel() === 0
      });
      return
      let diagram = fromNode.diagram;
      // diagram.startTransaction("Add State");
      // get the node data for which the user clicked the button

      // create a new "State" data object, positioned off to the right of the fromNode
      let p = fromNode.location.copy();
      p.x += diagram.toolManager.draggingTool.gridSnapCellSize.width;
      let toData = null
      let model = diagram.model;
      //添加倒数第二个节点  进行节点连线至最后节点
      toData = Object.assign({
        key: (Math.random() * (36 - 1) + 1).toString(32).substr(3, 6),
        parent: fromData.key,
        parentName: fromData.name,//上级名称
        name: "一个名字"
        // loc: go.Point.stringify(p),
      });

      // add the new node data to the model
      model.addNodeData(toData);
      //  添加连线
      this.setLinks(model, fromData, toData)
      // 选中新增的node
      let newnode = diagram.findNodeForData(toData);

      diagram.select(newnode);


    },
    /**
     * 添加节点
     * @param {*} nodeData 节点数据
     */
    addMindmapNodeData(nodeData) {
      const model = this.myDiagram.model;
      // 添加节点
      model.addNodeData(nodeData);
      // 添加连线

      let linkdata = {
        from: this.activeMindmapNodeData.key,
        to: nodeData.key
      };
      // 添加连线
      model.addLinkData(linkdata);
      // 选中新增的node
      const newnode = this.myDiagram.findNodeForData(nodeData);
      // 选中新增节点
      this.myDiagram.select(newnode);
    },
    /**
   * 编辑
   */
    editNode(e, obj, type) {
      const nodeData = obj.part.data
      console.log('编辑', obj.part.data)
      this.$emit('updateNode', {
        isTreeRoot: obj.part.adornedPart.findTreeLevel() === 0,
        nodeData
      })

    },
    /**
       * 更改节点属性
       * @param {*} data 当前属性值
       * @param {*} type 节点属性
      */
    updateNodeData(data) {
      let model = this.myDiagram.model;
      model.updateTargetBindings(Object.assign({}, this.activeMindmapNodeData, data));
      // 修改节点data属性.
      // this.myDiagram.model.setDataProperty(thisemp.data, 'entity', data.entity);
    },
    // 自适应
    zoomToFit() {
      this.myDiagram.commandHandler.zoomToFit();
      this.getData()
    },
    // 移动至某个节点
    scrollToView() {
      this.myDiagram.commandHandler.scrollToPart(this.myDiagram.findNodeForKey(1));
    },
    //获取数据
    getData() {
      console.log(this.myDiagram.model.toJson())
    },

  },
  mounted() {
    //初始化gojs
    this.initGojs()
  },
  beforeDestroy() {
    this.timer && window.clearTimeout(this.timer)
  }
}
</script>
<style   lang="less" scoped>
.five-main {
  height: 100%;
  width: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-direction: row;
  overflow: hidden;
}
.gojs {
  flex: 1;
  height: 100%;
}
.gojs-tool-bar {
  position: absolute;
  bottom: 10px;
  right: 10px;
  z-index: 100;
  text-align: right;
  i {
    font-size: 30px;
    cursor: pointer;
  }
  .gojs-overview-container {
    width: 300px;
    height: 200px;
    border: 1px solid #ececec;
  }
}
</style>