<script src="../../../../MyCode/Family/js/Family.js"></script>
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
        mine: null,
        datas: [],
        node_w: 50, //矩形的宽度
        node_h: 70, //矩形的高度
        default_w: 30, //间距左右
        default_h: 30, //山下间距
        paths: {},
        screens: [],
        transform: {
          x: 0,
          y: 0
        },
        zoom: null,
        offset_w: null,
        offset_h: null,
        centerId: '',
        svg: null,
        //数据依赖格式说明，男性节点生成妻子，儿子，女儿关系，女性节点生成丈夫，儿子，女儿关系
        nodes: [
          {id: '0', memberId: '0', name: '张三', memberName: '张三', sex: 1, type_name: '本人', algebra: '2'},
          {id: '1', memberId: '1', name: '张父', memberName: '张父', sex: 1, type_name: '父亲', algebra: '1'},
          {id: '2', memberId: '2', name: '张母', memberName: '张母', sex: 1, type_name: '母亲', algebra: '1'},
          {id: '3', memberId: '3', name: '张三妻', memberName: '张三妻', sex: 2, type_name: '伴侣', algebra: '2'},
          {id: '4', memberId: '4', name: '张大', memberName: '张大', sex: 1, type_name: '儿子', algebra: '3'},
          {id: '5', memberId: '5', name: '张小', memberName: '张小', sex: 1, type_name: '儿子', algebra: '3'},
          {id: '6', memberId: '6', name: '张小妻', memberName: '张小妻', sex: 2, type_name: '儿子伴侣', algebra: '3'},
          {id: '7', memberId: '7', name: '张小小', memberName: '张小小', sex: 1, type_name: '孙子', algebra: '4'},
          {id: '8', memberId: '8', name: '张三妻父', memberName: '张三妻父', sex: 1, type_name: '伴侣父亲', algebra: '1'},
          {id: '9', memberId: '9', name: '张三妻母', memberName: '张三妻母', sex: 2, type_name: '伴侣母亲', algebra: '1'},
          {id: '10', memberId: '10', name: '张三妻弟', memberName: '张三妻弟', sex: 1, type_name: '伴侣弟弟', algebra: '2'}
        ],
        //1父亲,2母亲,3妻子,4丈夫,5儿子,6女儿
        links: [
          {srcId: '0', dst: '0', toId: '1', type: 5}, // 张三父子
          {srcId: '0', dst: '0', toId: '2', type: 5}, // 张三母子
          {srcId: '0', dst: '0', toId: '3', type: 4}, // 张三伴侣
          {srcId: '0', dst: '0', toId: '4', type: 1}, // 张三子女
          {srcId: '0', dst: '0', toId: '5', type: 1}, // 张三子女
          {srcId: '3', dst: '3', toId: '4', type: 2}, // 张三妻子女
          {srcId: '3', dst: '3', toId: '5', type: 2}, // 张三妻子女
          {srcId: '1', dst: '1', toId: '2', type: 4}, // 张父伴侣
          {srcId: '5', dst: '5', toId: '6', type: 4}, // 张小伴侣
          {srcId: '5', dst: '5', toId: '7', type: 1}, // 张小子女
          {srcId: '6', dst: '6', toId: '7', type: 2}, // 张小妻子女
          {srcId: '3', dst: '3', toId: '8', type: 6}, // 张三妻父
          {srcId: '8', dst: '8', toId: '9', type: 4}, // 张三妻父伴侣
          {srcId: '8', dst: '8', toId: '10', type: 1} // 张三妻父子女
        ],
        drag: false
      };
    },
    filters: {
      toCn(src) {
        let res = ['父子', '母子', '配偶', '子女'];
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
        let height = 800;
        d3.select('.testd3').selectAll('*').remove();
        //创建画布
        // 画中心点
        let svg = d3.select('.testd3').append('svg')
          .attr('width', width)
          .attr('height', height)
          .style("padding", "10px")
          .style("background", "#f5f5f5");
        // .append('g')
        // .attr('transform', 'translate(40,0)');
        //
        _that.zoom = d3.zoom()
        //参数  是或否  1不放大  2放大
          .scaleExtent([1, 2])
          .on("zoom", _that.zoomed);
        svg = svg.append("g").attr("transform", "translate(" + 0 + "," + 0 + ")").call(_that.zoom);
        //内部滑动画布
        var rect = svg.append("rect")
          .attr("width", 600)
          .attr("height", 1280)
          .style("fill", "none")
          .style("pointer-events", "all");

        _that.svg = svg.append("g")
          .attr('style', 'background:red');
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
        //-------------------------------------
        //图表分组
        //计算最大深度
        var max_algebra = 0;
        _that.nodes.forEach(function (item) {
          if (typeof item.algebra != "undefined") {
            if (item.algebra > max_algebra) {
              max_algebra = item.algebra;
            }
          }
        })
        var algebra = max_algebra;
        do {
          _that.compute_points_algebra(algebra);
        } while (algebra-- > 1)


        var algebra = max_algebra;

        do {
          _that.compute_links_algebra(algebra);

        } while (algebra-- > 1)

        var offset_x = this.totalW();
        //高吊线图底部  到屏幕顶部高度
        var offset_y = 667;
        _that.svg.attr("transform", "translate(" + offset_x + "," + offset_y + ")")
        console.log(this.links)
        _that.drawLinks(_that.svg, _that.paths);
        _that.drawNodes(_that.svg, _that.nodes);
        _that.transform.x = _that.totalW();
        //高吊线图底部  到屏幕顶部高度
        _that.transform.y = 667;
        _that.comPosition();
        //根据id 显示的正中间的位置
        _that.moveNode(_that.centerId, 100, 0);
        //-------------------------------------

        // // var drag = d3.behavior.drag()
        // var drag = d3.drag()
        //   .on('drag', function (d) {
        //     d3.select(this)
        //       .attr('transform', 'translate(' + (d3.event.x - 30) + ',' + (d3.event.y - 30) + ')')
        //       .append('rect')
        //       .attr('x', d.x = d3.event.x - 30)
        //       .attr('y', d.y = d3.event.y - 30);
        //     // 线条
        //     svg.selectAll(`.link-${d.id}`).attr('d', function (dd) {
        //       return _that.getPath(d, dd);
        //     });
        //   });
        // _that.drag = drag;
        //
        // this.srcNode = row;
        // // 查找中心点连线关系 srcId = 0
        // let srcId = row.id;
        // let srcNode = this.nodes.filter(n => n.id === srcId)[0];
        // // 获取与此节点关系点
        // let links = this.links.filter(l => l.srcId === srcId);
        // let otherlinks = this.links.filter(l => l.toId === srcId).map(l => {
        //   let link = {};
        //   if (l.type === 3) { // 子女 -> 父子
        //     link.type = 0;
        //   } else if (l.type === 0) { // 父女 -> 子女
        //     link.type = 3;
        //   } else {
        //     link.type = l.type;
        //   }
        //   link.srcId = l.toId;
        //   link.toId = l.srcId;
        //   return link;
        // });
        // links.push(...otherlinks);
        // // 计算节点坐标
        // let nodes_ = this.convert(srcNode, links);
        // // 画线
        // this.drawLinks(svg, links);
        // // 画点
        // console.log('nodes_', nodes_);
        // this.drawNodes(svg, nodes_);
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
          if (l.type === 2) { // 配偶
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
        if (stas.type === 0) { // 父子
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
        if (stas.type === 2 || stas.type === 1) { // 配偶
          let w_ = move.x > x ? width_ : -width_;
          let padding = move.x > x ? -width_ / 2 : width_ * 1.5;
          if (flag) {
            path = `M${x + width_ / 2 + w_ / 2} ${y + width_ / 2}
                L${move.x + padding} ${y + width_ / 2}
                L${move.x + width_ / 2 - w_ / 2} ${move.y + width_ / 2}`;
          } else {
            path = `M${x + width_ / 2 + w_ / 2} ${y + width_ / 2}
                L${move.x + padding} ${y + width_ / 2}
                L${move.x + width_ / 2} ${move.y + width_ / 2}`;
          }
        }
        if (stas.type === 3) { // 子女
          if (flag) {
            path = `M${x + width_ / 2} ${y + width_}
            L${x + width_ / 2} ${y - width_ / 2}
            L${move.x + width_ / 2} ${move.y + width_ * 1.5}
            L${move.x + width_ / 2} ${move.y + width_}`;
          } else {
            path = `M${x + width_ / 2} ${y + width_}
            L${x + width_ / 2} ${y + width_ * 1.5}
            L${move.x + width_ / 2} ${move.y - width_ / 2}
            L${move.x + width_ / 2} ${move.y + width_}`;
          }
        }
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
        this.drawNodes(svg, nodes2);
      },
      drawNodes(svg, nodes) {
        nodes.forEach(n => {
          this.drawNode(svg, n);
        });
      },
      //添加节点- 原
      /*drawNode(svg, node) {
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
      },*/
      //添加节点- 新
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
        // 矩形
        node_.append('rect')
          .attr('width', _that.node_w)
          .attr('height', _that.node_h)
          .attr('x', 0)
          .attr('y', 0)
          .attr('rx', 3)
          .attr('ry', 3)
          .attr('style', (d) => {
            // if(d.memberId == _that.centerId){
            if (d.memberId === 0) {
              return 'fill:rgb(255,255,255);stroke:#c3141c;stroke-width:2;'; //中心
            } else if (d.sex === 1) {
              return 'fill:rgb(255,255,255);stroke:rgb(76,141,171);stroke-width:1;'; //男同志
            } else {
              return 'fill:rgb(255,255,255);stroke:rgb(244,121,131);stroke-width:1;'; //女同志
            }
          });
        //矩形中添加三角形
        node_.append('polygon')
          .data([node])
          .attr('points', "0.4,0.4 25,0 0,25")
          .attr('style', (d) => {
            if (d.sex === 1) {
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
          .style('font-size', '6px')
          .style('display', 'block')
          .text(function (d) {
            return d.type_name;
          });
        node_.append('text')
          .attr('dx', function (d) {
            if (d.memberName) {
              return (_that.node_w - 2 * 10 + 20) / 2;
            }
          })
          .attr('dy', function (d) {
            return _that.node_h - 10;
          })
          .style('text-anchor', function (d) {
            return 'middle';
          })
          .style('fill', '#000000')
          .style('padding', '1px 10px')
          .style('background', 'red')
          .style('display', 'block')
          .style('font-size', '10px')
          .text(function (d) {
            return d.memberName.substring(0, 5)
          });
      },
      // 添加链接 - 原
      /*drawLinks(svg, links) {
        let that = this;
        links.forEach(l => {
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
            .attr('style', function () {
              return 'stroke:#FFCC33;fill:none;';
            });
        });
      },*/
      // 添加链接 - 新
            drawLinks(svg, links) {
              let that = this;
              for (var l in links) {
                svg.selectAll(`.link-${l.memberId}`)
                  .data([links[l]])
                  .enter()
                  .append('path')
                  .attr('class', d => {
                    return `link-${d.memberId}`;
                  })
                  .attr('d', function (d) {
                    let node = that.nodes.filter(n => n.id === d.toId)[0];
                    return d.path;
                  })
                  .attr('style', function () {
                    return 'stroke:#FFCC33;fill:none;';
                  });
              }
            },
      comPosition() {
        var _that = this
        var min_pos = 0;
        _that.nodes.forEach(function (item) {
          if (typeof item.x != "undefined") {
            if (item.x < min_pos) {
              min_pos = item.x;
            }
          }
        })
        //				console.log(min_pos)
        _that.nodes.forEach(function (item) {
          item.pos_x = item.x - min_pos
        })
      },
      totalW() {
        var _that = this
        var min_algebra = 9999;
        _that.nodes.forEach(function (item) {
          if (typeof item.algebra != "undefined") {
            if (item.algebra < min_algebra) {
              min_algebra = item.algebra;
            }
          }
        })
        var algebra = min_algebra;
        var current_nodes = _that.findDepthNodes(algebra, -1);
        var total_w = 0;
        current_nodes.forEach(function (item) {
          total_w += item.w;
        })
        return total_w + _that.default_w
      },
      zoomed() {
        var _that = this;
        var x = d3.event.transform.x + _that.transform.x;
        var y = d3.event.transform.y + _that.transform.y;
        _that.svg.attr("transform", `translate(${x},${y}),scale(${d3.event.transform.k})`);
      },
      addMine: function () {
        this.$store.state.gx = 'own'
        this.$router.push({
          path: '/add'
        })
      },
      moveNode(memberId, x, y) {
        var _that = this;
        var itemc = _that.nodes.find(function (item) {
          return item.memberId == memberId
        })


        if (typeof itemc == "undefined") {
          return;
        }

        var offset_x = x - itemc.pos_x;
        var offset_y = y - itemc.y;
        _that.transform.x += offset_x;
        _that.transform.y += offset_y;


        _that.svg.attr("transform", `translate(${_that.transform.x},${_that.transform.y})`);


      },
      compute_links_algebra(algebra) {
        //				console.log("开始处理algebra:"+algebra)
        var _that = this;
        //寻找女性节点
        var current_nodes = _that.findDepthNodes(algebra, 2);
        current_nodes.forEach(function (parent) {
          // console.log("查找memberId"+parent.memberId+"的丈夫:");
          //寻找丈夫节点
          var husbands = _that.findTypeNodes(parent, 4)
          //找到了丈夫节点
          if (husbands.length > 0) {
            husbands.forEach(function (husband, index) {
              //路径坐标
              var path = {
                "memberId": parent.memberId + "-" + husband.memberId,
                "path": `M${parent.x + _that.node_w} ${parent.y + _that.node_h / 2}
                                L${husband.x} ${parent.y + _that.node_h / 2}`
              }
              _that.$set(_that.paths, parent.memberId + "-" + husband.memberId, path);
            });
          } //什么也不做
        })

        //寻找男性节点
        var current_nodes = _that.findDepthNodes(algebra, 1);
        current_nodes.forEach(function (parent) {
          //					console.log("查找memberId"+parent.memberId+"的妻子:");
          //寻找妻子节点
          var wifes = _that.findTypeNodes(parent, 3)
          //找到了妻子节点
          if (wifes.length > 0) {
            wifes.forEach(function (wife, index) {
              //路径坐标
              var path = {
                "memberId": parent.memberId + "-" + wife.memberId,
                "path": `M${parent.x + _that.node_w} ${parent.y + _that.node_h / 2}
								L${wife.x} ${parent.y + _that.node_h / 2}`
              }
              _that.$set(_that.paths, parent.memberId + "-" + wife.memberId, path);
            });
          } //什么也不做

          //寻找儿子节点
          var sons = _that.findTypeNodes(parent, 5);
          //找到了儿子节点
          if (sons.length > 0) {
            sons.forEach(function (son, index) {
              //路径坐标
              var path = {
                "memberId": parent.memberId + "-" + son.memberId,
                "path": `M${parent.x + _that.node_w} ${parent.y + _that.node_h / 2}
								L${son.x + _that.node_w / 2} ${parent.y + _that.node_h / 2}
								v${son.y - parent.y + _that.node_h / 2}`
              }
              _that.$set(_that.paths, parent.memberId + "-" + son.memberId, path);
            });
          } //什么也不做

          //寻找女儿节点
          var sons = _that.findTypeNodes(parent, 6);
          //找到了儿子节点
          if (sons.length > 0) {
            sons.forEach(function (son, index) {
              //路径坐标
              var path = {
                "memberId": parent.memberId + "-" + son.memberId,
                "path": `M${parent.x + _that.node_w} ${parent.y + _that.node_h / 2}
								L${son.x + _that.node_w / 2} ${parent.y + _that.node_h / 2}
								v${son.y - parent.y + _that.node_h / 2}`
              }
              _that.$set(_that.paths, parent.memberId + "-" + son.memberId, path);
            });
          } //什么也不做

        })
        //55555

      },
      compute_points_algebra(algebra) {
        //				console.log("开始处理algebra:"+algebra)
        var _that = this;
        //寻找女性节点
        var current_nodes = _that.findDepthNodes(algebra, 2);
        current_nodes.forEach(function (parent) {
          //寻找丈夫节点
          var husbands = _that.findTypeNodes(parent, 4)
          if (husbands.length > 0) {
            //排序节点
            husbands = _that.sortKey(husbands, 'paihang');
            husbands.forEach(function (husband, index) {
              if (typeof husband.x == "undefined") {
                husband['y'] = 0;
                husband['x'] = 0;
              }

              var child_allw = 0;
              var child_w = 0;

              if (index == 0) {
                //妻子起始
                husband['x'] = parent['x']
              } else {
                //妻子的丈夫起始
                husband['x'] = husbands[index - 1].x
              }
              husband['x'] += _that.node_w;

              child_allw = _that.default_w
              husband['x'] += child_allw;
              return husband;
            })
            //节点总宽度
            parent['w'] = husbands[husbands.length - 1].x - husbands[0].x + 2 * _that.node_w + _that.default_w
          } else {
            //parent['w'] = _that.default_w
          }
        })
        //寻找男性节点
        var current_nodes = _that.findDepthNodes(algebra, 1);
        current_nodes.forEach(function (parent) {
          //					console.log("查找memberId"+parent.memberId+"的妻子:");
          //寻找妻子节点
          var wifes = _that.findTypeNodes(parent, 3)

          //取得妻子关联孩子节点
          var wifes_childs = [];
          wifes.forEach(function (wife, index) {
            //寻找孩子节点
            var son = _that.findTypeNodes(wife, 5);
            var daughter = _that.findTypeNodes(wife, 6);
            var childs = son.concat(daughter);
            wifes_childs = wifes_childs.concat(childs);
          })

          //取得父亲关联孩子节点
          var parent_childs = [];
          var son = _that.findTypeNodes(parent, 5);
          var daughter = _that.findTypeNodes(parent, 6);
          var childs = son.concat(daughter);
          parent_childs = parent_childs.concat(childs);
          //不等的情况表示有离散孩子节点
          if (parent_childs.length != wifes_childs.length) {
            //取得差集
            // var other_childs = [...new Set(parent_childs)].filter(x => [...new Set(wifes_childs)].every(y => y.memberId !== x.memberId));
            var other_childs = parent_childs.filter(x => !new Set(wifes_childs).has(x));
            //计算位置及相对偏移
            if (other_childs.length > 0) {
              //排序节点
              other_childs = _that.sortKey(other_childs, 'paihang');
              //孩子节点总宽度
              var child_all_w = 0;
              //孩子节点宽度
              var child_w = 0;
              var child_ws = [];
              other_childs.forEach(function (child, index) {
                if (typeof child.w != "undefined") {
                  child_w = child.w;
                } else {
                  //女儿节点只有自己
                  child_w = _that.node_w
                }

                //后置节点填充空白
                if (index > 0) {
                  child_all_w += child_w + _that.default_w
                } else {
                  child_all_w += child_w
                }

                child_ws.push(child_all_w);

              })
              //最左侧位置
              if (wifes.length > 0) {
                var start_x = parent.x - child_all_w
              } else {
                var start_x = parent.x + (_that.node_w - child_all_w) / 2
              }

              other_childs.forEach(function (child, index) {
                //计算子树
                _that.cnodes = {};
                _that.getChildNodes(child);
                _that.$set(_that.cnodes, child.memberId, Object.assign({}, child))
                //平移子树-含实际偏移量
                if (index == 0) {
                  _that.move(_that.cnodes, child.left_offset + start_x - child.x, parent.y + _that.default_h + _that.node_h -
                    child.y);
                } else {
                  _that.move(_that.cnodes, child.left_offset + start_x + _that.default_w + child_ws[index - 1] - child.x,
                    parent.y + _that.default_h + _that.node_h - child.y);
                }
              })
              parent['w'] = child_all_w
              //设置偏移量
              if (wifes.length > 0) {
                parent.left_offset = child_all_w
              } else {
                parent.left_offset = (child_all_w - _that.node_w) / 2
              }

            } else {
              parent['w'] = _that.node_w
            }
          }

          //找到了妻子节点
          if (wifes.length > 0) {
            //排序节点
            wifes = _that.sortKey(wifes, 'paihang');
            //循环妻子节点
            wifes.forEach(function (wife, index) {
              //初始化检查
              if (typeof wife.x == "undefined") {
                wife['y'] = 0;
                wife['x'] = 0;
              }

              //起始计算位置
              var start_x = 0;
              if (index == 0) {
                //丈夫起始,偏移修正
                start_x = parent['x']
              } else {
                //丈夫的妻子起始
                start_x = wifes[index - 1].x
              }

              //寻找孩子节点
              var son = _that.findTypeNodes(wife, 5);
              var daughter = _that.findTypeNodes(wife, 6);
              var childs = son.concat(daughter);
              if (childs.length > 0) {
                //排序节点
                childs = _that.sortKey(childs, 'paihang');
                //孩子节点总宽度
                var child_all_w = 0;
                //孩子节点宽度
                var child_w = 0;
                //								console.log("child forEach")
                childs.forEach(function (child, index) {
                  //									console.log(child)
                  if (typeof child.w != "undefined") {
                    child_w = child.w;
                  } else {
                    //女儿节点只有自己
                    child_w = _that.node_w
                  }

                  //后置节点填充空白
                  if (index > 0) {
                    child_all_w += child_w + _that.default_w
                  } else {
                    child_all_w += child_w
                  }

                  //计算子树
                  _that.cnodes = {};
                  _that.getChildNodes(child);
                  _that.$set(_that.cnodes, child.memberId, Object.assign({}, child))
                  //平移子树-含实际偏移量
                  _that.move(_that.cnodes, child.left_offset + start_x + _that.node_w + child_all_w - child_w - child.x,
                    wife.y + _that.default_h + _that.node_h - child.y);

                })
                //妻子的起点位置
                wife['x'] += start_x + _that.node_w + child_all_w;
              } else {
                //没有孩子时
                //妻子的起点位置
                wife['x'] += start_x + _that.node_w + _that.default_w;
              }

            })

            //当前节点占位宽度
            parent['w'] = parent.left_offset + wifes[wifes.length - 1].x - parent.x + _that.node_w
          }


          return parent;
        })
      },
      move(nodes, x, y) {
        var _that = this;
        for (var memberId in nodes) {
          var itemc = _that.nodes.find(function (item) {
            return item.memberId == memberId
          })
          if (itemc) {
            itemc['x'] += x;
            itemc['y'] += y;
          }
          //
        }
      },
      getChildNodes(node) {
        var _that = this;
        var itemc = _that.nodes.find(function (item) {
          return item.memberId == node.memberId
        })
        var son = _that.findTypeNodes(itemc, 5);
        var daughter = _that.findTypeNodes(itemc, 6);
        var wife = _that.findTypeNodes(itemc, 3);
        var husbands = _that.findTypeNodes(itemc, 4);
        var childs = son.concat(daughter);
        var childs = childs.concat(wife);
        var childs = childs.concat(husbands);
        if (childs.length > 0) {
          childs.forEach(function (child) {
            _that.$set(_that.cnodes, child.memberId, Object.assign({}, child))
            _that.getChildNodes(child)
          })
        }
      },
      sortKey(array, key) {
        return array.sort(function (a, b) {
          var x = a[key];
          var y = b[key];
          return ((x < y) ? -1 : (x > y) ? 1 : 0)
        })
      },
      findDepthNodes(algebra, sex) {
        var _that = this;

        var items = _that.nodes.filter(function (item, index, arr) {
          if (typeof item.x == "undefined") {
            item.x = 0;
            item.y = 0;
            item.left_offset = 0;
          }
          if (typeof item.w == "undefined") {
            item.w = _that.node_w;
          }
          if (sex !== -1) {
            if (typeof item.algebra != "undefined" && typeof item.sex != "undefined") {
              return (item.algebra == algebra && item.sex == sex)
            } else {
              return false;
            }
          } else {
            return (item.algebra == algebra)
          }
        })

        return items;
      },
      findTypeNodes(node, sex) {
        var _that = this;
        //1父亲,2母亲,3妻子,4丈夫,5儿子,6女儿
        var links = _that.links.filter(function (item, index, arr) {
          return (item.toId == node.memberId && item.type == sex)
        })
        var nodes = [];
        links.forEach(function (link) {
          nodes.push(_that.nodes.find(function (item) {
            return link.srcId == item.memberId
          }))
        })
        return nodes;
      },
      mounted() {
      }
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
