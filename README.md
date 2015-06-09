# earthquake-viz
earthquake visualize
http://www.infoq.com/cn/articles/visualization-of-the-global-seismic-system
GIS(Geographic Information System) 在可视化方面，无疑是有很多优势的，相比于传统的表格或者图表，它更容易将读者快速的带入到具体场景中，并更好的理解图形背后的含义。

很多信息事实上都和地理位置有关，比如关键客户的分布图，货源/仓库在地理上的分布等，当这些信息以可视化的方式展现在地图上时，我们可以获得很多之前无法看到的信息，而这些信息可以帮助我们在未来做出更合理的决策。

本文将使用开源工具OpenLayers做一个实例，并在这个过程中详细讨论GIS背后的一些技术细节。在这个例子中，我们将会把地球上的地震统计信息用可视化的方式，直观的展现在地图上。

例子的最终运行结果如下：



图中的绿色点表示小于里氏3级的地震，红色的点表示大于里氏3级的地震。

Web GIS简介

Web GIS是将传统的GIS与Web集成起来，借助于Web的高可用性，Web GIS可以使地图服务本身更容易被人们获得。成熟的Web GIS产品已经有很多，比如Google Maps，Bing Maps等，这些GIS产品已经很早就深入了我们的日常生活，并为我们的学习工作带来了众多的便利，比如去一个陌生的城市出差时找到预定的酒店，查找离你目前所处位置最近的银行等等。

那么我们看到的网页上很漂亮的地图是怎么产生的呢？它们又是如何被渲染到页面上的呢？

地图的渲染方式

几乎所有我们看到的基于Web的地图，都是通过不同层次叠加出来的结果。GIS系统会将 河流，建筑物，街道，文本标签的信息分别存储与不同的层次上（物理上不同的文件/数据库中），这样做不但可以在编辑地图时更容易管理（比如表示街道的图层的更新频率肯定会高于表示河流的图层的更新频率 ），而且可以按需加载（只查看河流，街道，不要建筑物信息等），从而减少服务器上的负载。

地图服务器将不同的层次叠加，最终生成一张完整的地图。这个最终的地图包含了所有的信息，比如河流，建筑物等都会带上名称。而且每个层次都可以独立的配色，这样最终的地图就是我们见到的样子了：蓝色的水域，黄色的高速路，白色的建筑，绿色的公园等等。



图片来源（http://www.srh.noaa.gov/bmx/?n=gis）

地图在服务器端被渲染出来之后，尺寸一般会非常大。需要由专门的模块将这些大图切分成很多组的小图，这些小图被称之为瓦片（tile）。为了给不同缩放级别的客户端提供不同的图片，这些瓦片被精心的分成了多个组，每个组都有编号。如果地图支持18级的缩放，就会现有18个分组。当然分组好越靠后，分组中的瓦片越多。



服务器上的瓦片

而在客户端，当我们在Google Maps上拖动地图，或者放大某一个感兴趣的区域时，往往会看到一些灰色的块。这些灰色的块过几秒会被真实的地图替换掉。这是因为，我们不可能也不需要将GIS服务器上的地图完全加载到客户端呈现，而是分批获取。在获取地图瓦片的过程中，客户端会展现一些占位符，当真正的瓦片加载完成后，再完成替换。

比如当我们查看西安市地图时，前端的JavaScript脚本会根据缩放级别和目前浏览的区域发送少量的请求获取地图瓦片，然后再将这些瓦片拼成一副“完整”的地图。

WMS请求

WMS(Web Map Service)是一个基于HTTP的简单协议，客户端发送的请求中包含请求类型，地图的层次，边界等信息，服务器根据这个信息生成图片，并返回该图片：



Chrome中的一次WMS请求详情

当然，WMS本身支持多种类型的请求，最常见的就是GetMap。具体的细节大家可以参考OGC规范及具体服务器的实现。而对于后端的服务器来说，从请求中获取这些信息之后，会首先从数据库/数据文件中得到数据，并使用渲染引擎绘制图片，并最后将图片返回客户端。

地图图层的分类

图层可以分为矢量图层和栅格图层，栅格图片一般的来源为航拍，遥感等技术，本质上来说就是照片，不过由于拍摄角度，器材本身产生的失真等，这些照片需要经过校正才能使用。由于栅格图片本身是拍摄的结果，那么放到到一定范围之后，必然会产生模糊。栅格图片通常会附带一些具有特别含义的属性，比如降水量，人口数等。



图片来源（http://earthobservatory.nasa.gov/Features/FalseColor/page3.php）

而矢量图层是数学上的抽象，矢量图层只需要记录在某个特定的坐标系中，各个点，线，面的关系即可，它可以任意缩放而不至于失真。同样，矢量图层中的元素也可以附加一些属性。



OpenLayers简介

OpenLayers是一个开源的JavaScript库，主要用于将不同来源的地图图层集成在一起，从而产生有价值的应用程序。一个最简单的场景是，底图使用Google Maps提供的卫星图层，而在这个图层之上加入与业务相关的矢量层，帮助读者快速的理解图层背后的含义。

OpenLayers支持的数据格式

OpenLayers支持很多种的数据格式，如GML，GeoJSON，GPX等等，大部分的GIS服务器都支持生成这样的数据格式以供展现。

其中，GML，GPX，Atom等都是基于XML的文档，而GeoJSON则是基于JSON格式的。我们的这个例子主要关注GeoJSON格式，它相对而言更小巧，更加可读。

{
  "type": "Feature",
  "geometry": {
    "type": "Point",
    "coordinates": [125.6, 10.1]
  },
  "properties": {
    "name": "Dinagat Islands"
  }
}
一个GeoJSON中的节点包含三部分的信息：类型，几何要素和属性集。几何要素可以是点，线，面或者多点，多线，多面。而属性则可以包含任意复杂的信息，比如一个表示建筑物的多边形上的可以包含诸如绿化面积，降雨量，植物种类等信息。

数据来源

美国地理信息调查局是一个科学组织，他公开了很多地球上的灾难信息，比如对地震的统计，并提供编程接口。它公开的地震统计信息，包含全世界各地报告过的地震，以及全美所有检测到的地震，并以多种周期（小时，天，周，月等），多种格式（GeoJSON，KML，Atom等）公开，以便应用程序的开发者只用这些数据。

我们的这个例子中，将使用USGS提供的GeoJSON格式的数据，并通过OpenLayers的API将这些数据展现在地图上。

逐步实现

设置基本环境

我们将借助bower来安装所有的代码依赖。首先，我们需要bower将所有的包都安装在components目录下，这个可以通过在当前目录的.bowerrc文件中指定directory：

{
    "directory": "components"
}
然后运行bower安装jquery以及openlayers：

$ bower install jquery
$ bower install openlayers
通过bower安装OpenLayers之后，可以通过OpenLayers自带的build工具将所有的源码合并压缩为一个文件：

$ cd components/openlayers/build
$ ./build.py #将会在当前目录下生成一个OpenLayers.js的文件
$ mv OpenLayers.js ../
然后，创建一个简单的HTML文件，引用jquery.js和OpenLayers.js，以及我们的入口脚本app.js，本文所有的代码都只是修改这个文件。

<!DOCTYPE HTML>
<html>
    <head>
        <meta http-equiv="content-type" content="text/html; charset=utf-8" />
        <title>Earthquake distribution</title>
        <link rel="stylesheet" href="style.css" />
    </head>
    <body>
        <div id="container">
            <div id="map">
            </div>
        </div>
        <script src="components/jquery/jquery.js" type="text/javascript"></script>
        <script src="components/openlayers/OpenLayers.js" type="text/javascript"></script>
        <script src="app.js" type="text/javascript"></script>
    </body>
</html>
基本代码

一个最简单的OpenLayers应用，只需要7行代码：

$(function() {
    var map = new OpenLayers.Map("map");
    var osm = new OpenLayers.Layer.OSM();

    map.addLayers([osm]);
    map.zoomToMaxExtent();
});
这段代码在id为map的HTML元素创建了一个地图，这个地图上有一个叫OSM的层（即OpenStreetMap，一个开源，类似于维基百科的地图平台），并将地图缩小到边界范围（以获得最大的视野）:



生成矢量图层
使用OpenLayers来生成矢量图非常容易：

var geo = new OpenLayers.Layer.Vector("EarthQuake", {
    strategies: [new OpenLayers.Strategy.Fixed()],
    protocol: new OpenLayers.Protocol.HTTP({
        url: '/all_day.geojson',
        format: new OpenLayers.Format.GeoJSON({ignoreExtraDims: true})
    })
});
注意此处的all_day.geojson是从USGS网站上下载的，过去一天中世界各地的所有地震统计。

上边的代码创建了一个名称为EarthQuake的矢量层，strategies中的Fixed策略表示仅请求一次资源，然后缓存在前端，不再请求。protocol表明数据来源为all_day.geojson，格式为OpenLayers.Format.GeoJSON。由于USGS返回的地理信息除了经纬度还包含深度，而OpenLayers默认只处理经纬度的，因此需要此处的ignoreExtraDims来忽略那个额外的深度信息。



定制样式

虽然我们已经加上了新的层，也可以看到很多表示地震的点信息，但是并不能看出哪些地震是严重的，比如里氏3级以下的地震，几乎没有危害，可以标注成一种颜色；而更高震级的可以标记成另外一种颜色。

OpenLayers可以很容易的做到这个定制化:

var style = new OpenLayers.Style();

var ruleLow = new OpenLayers.Rule({
  filter: new OpenLayers.Filter.Function({
        evaluate: function(properties) {
            return properties.mag < 3.0;
        }
    }),
  symbolizer: {pointRadius: 3, fillColor: "green",
               fillOpacity: 0.5, strokeColor: "black"}
});

var ruleHigh = new OpenLayers.Rule({
  filter: new OpenLayers.Filter.Function({
        evaluate: function(properties) {
            return properties.mag >= 3.0;
        }
    }),
    symbolizer: {pointRadius: 5, fillColor: "red",
               fillOpacity: 0.7, strokeColor: "black"}
});

style.addRules([ruleLow, ruleHigh]);

geo.styleMap = new OpenLayers.StyleMap(style);
首先创建一个Style对象，为Style添加两条规则Rule，然后将Style对象包装成StyleMap并赋值给表示地震的矢量层earthquake。

对于规则ruleLow，我们定义了，当一个feature的属性值mag(震级)小于三的时候后，使用绿色的，半径为3px的小圆圈来表示。而ruleHigh则定义了当震级大于等于三的时候，用红色，半径为5px的圆圈来表示。



更高级的样式规则

OpenLayers有完善的样式规则模型，比如我们需要按等级生成不同的颜色标记：在里氏1到3级为绿色，3级到5级为黄色，大于5级为红色：

var rules = [
        new OpenLayers.Rule({
            filter: new OpenLayers.Filter.Function({
                evaluate: function(properties) {
                    return properties.mag < 3.0;
                }
            }),
            symbolizer: {
            pointRadius: 3, fillColor: "green",
            fillOpacity: 0.5, strokeColor: "black"
            }
        }),  
        new OpenLayers.Rule({
            filter: new OpenLayers.Filter.Function({
                evaluate: function(properties) {
                    return properties.mag >= 3.0 && properties.mag < 5.0;
                }
            }),
            symbolizer: {
            pointRadius: 5, fillColor: "orange",
            fillOpacity: 0.5, strokeColor: "black"
            }
        }),  
        new OpenLayers.Rule({
            filter: new OpenLayers.Filter.Function({
                evaluate: function(properties) {
                    return properties.mag >= 5.0;
                }
            }),
            symbolizer: {
            pointRadius: 7, fillColor: "red",
            fillOpacity: 0.5, strokeColor: "black"
            }
        }),
    ];


事件处理

虽然我们已经可以直观的根据震级不同而看到不同颜色的点，但是整个应用仍然没有多少意义：它不具备于用户的交互能力。我们需要添加上事件处理，当用户点击地图上的一个圆点的时候，应该看到一个更详细的窗口。

var selectControl = new OpenLayers.Control.SelectFeature(geo, {
    onSelect: onFeatureSelect,
    onUnselect: onFeatureUnselect
});

map.addControl(selectControl);
selectControl.activate();

function onFeatureSelect(feature) {
    var html = "<span>"+feature.attributes.title+"</span>";

    var popup = new OpenLayers.Popup.FramedCloud("popup",
            feature.geometry.getBounds().getCenterLonLat(),
            null,
            html,
            null,
            true
        );

    popup.panMapIfOutOfView = true;
    popup.autoSize = true;

    feature.popup = popup;

    map.addPopup(popup);
}

function onFeatureUnselect(feature) {
    map.removePopup(feature.popup);
    feature.popup.destroy();
    feature.popup = null;
}
我们在地图上添加了一个SelectFeature控件，并注册了回调函数：当矢量层中的矢量被选中之后，函数onFeatureSelect将被执行。我们可以在这个函数中添加对弹出窗口的控制。当onFeatureSelect执行时，OpenLayers会将当前的Feature传递进来，我们可以动态的取得震级，标题，链接等信息，并展现给最终用户。



完整的代码示例可以看这里。通过这个例子，我们知道了如何使用OpenLayers展现既有的数据源生成矢量图层，并了解了如何为这些图层应用不同的样式。最后，我们了解了如何为矢量图层中的特征注册事件处理器。

毫无疑问，我们最终的地图应用中对数据的展现，是其他传统展现方式无法比拟的。通过使用OpenLayers和公开的数据源，我们可以很容易的搭建起一些很有用的可视化应用，这些应用也在很多方面使得我们的生活更加便利。

感谢张凯峰对本文的审校和策划。
