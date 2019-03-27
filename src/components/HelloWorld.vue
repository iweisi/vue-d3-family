<template lang='html'>
  <div class='demo'>
    <div class='left'>
      <div class='left-top'>
        <el-table
          height='250'
          highlight-current-row
          @current-change="handleCurrentChange"
          :data='nodes'>
          <el-table-column
            prop='id'
            label='id'
            width='50'>
          </el-table-column>
          <el-table-column
            prop='name'
            label='姓名'>
          </el-table-column>
        </el-table>
      </div>
      <div class='left-bottom'>
        <el-table
          height='250'
          :data='links'>
          <el-table-column
            prop='srcId'
            label='源id'
            width='50'>
          </el-table-column>
          <el-table-column
            prop='toId'
            label='目标id'>
          </el-table-column>
          <el-table-column
            label='关系'>
            <template slot-scope='scope'>{{ scope.row.type | toCn }}</template>
          </el-table-column>
        </el-table>
      </div>
    </div>
    <div class='testd3' ref="testd3">
    </div>
  </div>
</template>

<script>
  import Vue from 'vue'
  import * as d3 from 'd3';
  import Element from 'element-ui';
  import 'element-ui/lib/theme-chalk/index.css';

  Vue.use(Element)

  let width_ = 60;
  export default {
    data() {
      return {
        srcNode: null,
        node_w: 50, //矩形的宽度
        node_h: 70, //矩形的高度
        default_w: 30, //间距左右
        default_h: 30, //山下间距
        paths: {},
        svg: null,
        nodes: [
          {id: '0', memberId:'0', name: '张三', sex: 1, type_name: '本人', algebra: '2'},
          {id: '1', memberId:'1',  name: '张父', sex: 1, type_name: '父亲', algebra: '1'},
          {id: '2', memberId:'2',  name: '张母', sex: 2, type_name: '母亲', algebra: '1'},
          {id: '3', memberId:'3',  name: '张三妻', sex: 2, type_name: '伴侣', algebra: '2'},
          {id: '4', memberId:'4',  name: '张大', sex: 1, type_name: '儿子', algebra: '3'},
          {id: '5', memberId:'5',  name: '张小', sex: 1, type_name: '儿子', algebra: '3'},
          {id: '6', memberId:'6',  name: '张小妻', sex: 2, type_name: '儿子伴侣', algebra: '3'},
          {id: '7', memberId:'7',  name: '张小小', sex: 1, type_name: '孙子', algebra: '4'},
          {id: '8', memberId:'8',  name: '张三妻父', sex: 1, type_name: '伴侣父亲', algebra: '1'},
          {id: '9', memberId:'9',  name: '张三妻母', sex: 2, type_name: '伴侣母亲', algebra: '1'},
          {id: '10',memberId:'10',  name: '张三妻弟', sex: 1, type_name: '伴侣弟弟', algebra: '2'}
        ],
        links: [
          {srcId: '0', toId: '1', type: 0}, // 张三父子
          {srcId: '0', toId: '2', type: 1}, // 张三母子
          {srcId: '0', toId: '3', type: 2}, // 张三伴侣
          {srcId: '0', toId: '4', type: 3}, // 张三子女
          {srcId: '0', toId: '5', type: 3}, // 张三子女
          {srcId: '3', toId: '4', type: 3}, // 张三妻子女
          {srcId: '3', toId: '5', type: 3}, // 张三妻子女
          {srcId: '1', toId: '2', type: 2}, // 张父伴侣
          {srcId: '5', toId: '6', type: 2}, // 张小伴侣
          {srcId: '5', toId: '7', type: 3}, // 张小子女
          {srcId: '6', toId: '7', type: 3}, // 张小妻子女
          {srcId: '3', toId: '8', type: 0}, // 张三妻父
          {srcId: '8', toId: '9', type: 2}, // 张三妻父伴侣
          {srcId: '8', toId: '10', type: 3} // 张三妻父子女
        ],
        drag: false
      };
    },
    filters: {
      toCn(src) {
        let res = ['父子', '母子', '伴侣', '子女'];
        return res[src];
      }
    },
    methods: {
      init() {
        var _that = this
        _that.nodes = [];
        _that.links = [];
        _that.paths = {};
        _that.current_nodes = {};
        _that.transform = {
          x: 0,
          y: 0
        }
      },
      handleCurrentChange(row) {
        let _that = this;
        // 设置画布
        let width = 1000;
        let height = 720;
        d3.select('.testd3').selectAll('*').remove();
        //创建画布
        // 画中心点
        let svg = d3.select('.testd3').append('svg')
          .attr('width', width)
          .attr('height', height)
          .style("padding", "10px")
          .style("background", "#f5f5f5")
          .append('g')
          .attr('transform', 'translate(40,0)');
        //
        // _that.zoom = d3.behavior.zoom()
        // //参数  是或否  1不放大  2放大
        //   .scaleExtent([1, 2])
        //   // .on("zoom", _that.zoomed);
        // svg = svg.append("g").attr("transform", "translate(" + 0 + "," + 0 + ")").call(_that.zoom)
        //内部滑动画布
        var rect = svg.append("rect")
          .attr("width", 600)
          .attr("height", 1280)
          .style("fill", "none")
          .style("pointer-events", "all");

        _that.svg = svg.append("g")
          .attr('style', 'background:red')
        // var drag = d3.behavior.drag()
        var drag = d3.drag()
          .on('drag', function (d) {
            d3.select(this)
              .attr('transform', 'translate(' + (d3.event.x - 30) + ',' + (d3.event.y - 30) + ')')
              .append('rect')
              .attr('x', d.x = d3.event.x - 30)
              .attr('y', d.y = d3.event.y - 30);
            // 线条
            svg.selectAll(`.link-${d.id}`).attr('d', function (dd) {
              return _that.getPath(d, dd);
            });
          });
        _that.drag = drag;

        this.srcNode = row;
        // 查找中心点连线关系 srcId = 0
        let srcId = row.id;
        let srcNode = this.nodes.filter(n => n.id === srcId)[0];
        // 获取与此节点关系点
        let links = this.links.filter(l => l.srcId === srcId);
        let otherlinks = this.links.filter(l => l.toId === srcId).map(l => {
          let link = {};
          if (l.type === 3) { // 子女 -> 父子
            link.type = 0;
          } else if (l.type === 0) { // 父女 -> 子女
            link.type = 3;
          } else {
            link.type = l.type;
          }
          link.srcId = l.toId;
          link.toId = l.srcId;
          return link;
        });
        links.push(...otherlinks);
        // 计算节点坐标
        let nodes_ = this.convert(srcNode, links);
        // 画线
        this.drawLinks(svg, links);
        // 画点
        console.log('nodes_', nodes_);
        this.drawNodes(svg, nodes_,0);
      },
      // 转换
      convert(srcNode, links) {
        let nodes = [];
        let map = new Map();
        // 子女个数
        let childsize = links.filter(l => l.type === 3).length;
        // 子女开始位置
        let start = -(childsize - 1) * width_;
        if (srcNode.x === undefined) { // 默认坐标
          srcNode.x = 200;
          srcNode.y = 200;
        }
        map.set(srcNode.id, srcNode);
        let {x, y} = srcNode;
        links.forEach(l => {
          if (l.type === 0) { // 父子
            map.set(l.toId, {x: x, y: y - width_ * 3, type: l.type});
          }
          if (l.type === 1) { // 母子
            map.set(l.toId, {x: x + width_ * 3, y: y - width_ * 3, type: l.type});
          }
          if (l.type === 2) { // 伴侣
            map.set(l.toId, {x: x + width_ * 4, y: y, type: l.type});
          }
          if (l.type === 3) { // 子女
            map.set(l.toId, {x: x + start, y: y + width_ * 4, type: l.type});
            start = start + width_ * 4;
          }
        });
        this.nodes.forEach(n => {
          let m = map.get(n.id);
          if (m) {
            n['x'] = n.x || m.x;
            n['y'] = n.y || m.y;
            n['type'] = m.type;
            nodes.push(n);
          }
        });
        return nodes;
      },
      getPath(move, stas) {
        // 获取对方坐标
        let srcId = stas.srcId;
        let flag = false;
        if (move.id === stas.srcId) {
          srcId = stas.toId;
          flag = true;
        }
        let path;
        // 源
        let {x, y} = this.nodes.filter(n => n.id === srcId)[0];
        if (stas.type === 0 || stas.type === 1) { // 父子
          if (flag) {
            path = `M${x + width_ / 2} ${y}
            L${x + width_ / 2} ${y + width_ * 1.5}
            L${move.x + width_ / 2} ${move.y - width_ / 2}
            L${move.x + width_ / 2} ${move.y}`;
          } else {
            path = `M${x + width_ / 2} ${y}
            L${x + width_ / 2} ${y - width_ / 2}
            L${move.x + width_ / 2} ${move.y + width_ * 1.5}
            L${move.x + width_ / 2} ${move.y}`;
          }
        }
        if (stas.type === 2) { // 伴侣
          let w_ = move.x > x ? width_ : -width_;
          let padding = move.x > x ? -width_ / 2 : width_ * 1.5;
          if (flag) {
            path = `M${x + width_ / 2 + w_ / 2} ${y + width_ / 2}
                L${move.x + padding} ${y + width_ / 2}
                L${move.x + width_ / 2 - w_ / 2} ${move.y + width_ / 2}`;
          } else {
            path = `M${x + width_ / 2 + w_ / 2} ${y + width_ / 2}
                L${move.x + padding} ${y + width_ / 2}
                L${move.x + width_ / 2  - 30} ${move.y + width_ / 2}`;
          }
        }
        if (stas.type === 3) { // 子女
          if (flag) {
            path = `M${x + width_ / 2} ${y + width_ - 60}
            L${x + width_ / 2} ${y - width_ / 2}
            L${move.x + width_ / 2} ${move.y + width_ * 1 - 60}
            L${move.x + width_ / 2 } ${move.y + width_ - 60}`;
          } else {
            path = `M${x + width_ / 2} ${y + width_}
            L${x + width_ / 2} ${y + width_ * 1.5}
            L${move.x + width_ / 2} ${move.y - width_ / 2 - 60}
            L${move.x + width_ / 2 } ${move.y + width_- 60}`;
          }
        }
        // console.log(move.name+path);
        return path;
      },
      nodeClick(svg, d) {
        let links = this.links.filter(l => l.srcId === d.id);
        if (links.length === 0) {
          this.$message('没有关联数据');
          return;
        }
        let nodes2 = this.convert(d, links);
        this.drawLinks(svg, links);
        this.drawNodes(svg, nodes2,d.memberId);
      },
      drawNode(svg, node) {
        var _that = this;
        let node_ = svg.selectAll(`.node-${node.id}`)
          .data([node])
          .enter()
          .append('g')
          .attr('class', `node-${node.id}`)
          .attr('transform', function (d) {
            return 'translate(' + (d.x) + ',' + (d.y) + ')';
          })
          .on('dblclick', (d) => {
            this.nodeClick(svg, d);
          })
          .on('mouseover', function (d) {
            d3.select(this).select('rect')
              .attr('stroke', '#FFCC33')
              .attr('stroke-width', 3); // 设置边框
          })
          .on('mouseout', function (d) {
            d3.select(this).select('rect')
              .attr('stroke-width', 0); // 取消边框
          })
          .call(this.drag);
        node_.append('rect')
          .attr('width', 60)
          .attr('height', 60)
          .attr('x', 0)
          .attr('y', 0)
          .attr('rx', 3)
          .attr('ry', 3)
          .attr('style', (d) => {
            if (d.id === _that.srcNode.id) {
              return 'fill:rgb(255,255,255);stroke:#c3141c;stroke-width:2;'; //中心
            } else if (d.sex === 1) {
              return 'fill:rgb(255,255,255);stroke:rgb(76,141,171);stroke-width:1;';
            } else {
              return 'fill:rgb(255,255,255);stroke:rgb(244,121,131);stroke-width:1;';
            }
          });
        //矩形中添加三角形
        node_.append('polygon')
          .data([node])
          .attr('points', "0.4,0.4 25,0 0,25")
          .attr('style', (d) => {
            if (d.sex == 1) {
              return 'fill-opacity:0.9;fill:rgb(76,141,171);'
            } else {
              return 'fill-opacity:0.9;fill:rgb(244,121,131);'
            }
          });
        node_.append('text')
          .attr('x', function (d) {
            return 0;
          })
          .attr('y', function (d) {
            return 0;
          })
          .attr('dx', function (d) {
            return 0;
          })
          .attr('dy', function (d) {
            return 14;
          })
          .attr('transform', function (d) {
            return 'rotate(-45)';
          })
          .style('text-anchor', function (d) {
            return 'middle';
          })
          .style('fill', '#ffffff')
          .style('padding', '1px 10px')
          .style('background', 'red')
          .style('display', 'block')
          .style('font-size', '10px')
          .text(function (d) {
            return d.type_name;
          });
        node_.append('text')
          .attr('dx', function (d) {
            return 30;
          })
          .attr('dy', 30)
          .style('text-anchor', function (d) {
            return 'middle';
          })
          .style('fill', '#000000')
          .style('padding', '1px 10px')
          .style('background', 'red')
          .style('display', 'block')
          .style('font-size', '10px')
          .text(function (d) {
            return d.name;
          });
        // this.nodeClick(svg,initNode);
      },
      drawNodes(svg, nodes,shield) {
        var _that = this;
        nodes.forEach(n => {
          _that.drawNode(svg, n);
        });
        //TODO 循环nodes 然后调用nodeclick方法
        nodes.forEach(n => {
          if(n.memberId != shield){
            this.nodeClick(svg,n);
          }
        });
      },
      // 添加链接 - 原
      drawLinks(svg, linksData) {
        let that = this;
        linksData.forEach(l => {
          let classFlag = l.srcId > l.toId ? `${l.srcId}-${l.toId}` : `${l.toId}-${l.srcId}`;
          svg.selectAll(`.link-${classFlag}`)
            .data([l])
            .enter()
            .append('path')
            .attr('class', d => {
              return `link-${d.srcId} link-${d.toId} link-${classFlag}`;
            })
            .attr('d', function (d) {
              let node = that.nodes.filter(n => n.id === d.toId)[0];
              return that.getPath(node, d);
            })
            .attr("stroke-dasharray",function(d,i){
              return "5,5";
            })
            .attr('style', function () {
              return 'stroke:#FFCC33;fill:none;';
            });
        });
      }
    },
    mounted() {
    }
  };
</script>

<style lang='css'>
  [class^=link] {
    fill: none;
    stroke: #ccc;
    stroke-width: 1.5px;
  }

  .demo {
    width: 100%;
    height: 100%;
  }

  .left {
    float: left;
    width: 30%;
  }

  .testd3 {
    float: left;
    width: 70%;
  }
</style>
