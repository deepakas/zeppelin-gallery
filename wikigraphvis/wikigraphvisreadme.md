 
# Zeppelin example on Graph Visualisation using wikipedia webanalytics data - D3 Force Layout and Sankey Chart 

### Zeppelin json notebook which contains the code is here. 
https://github.com/deepakas/zeppelin-gallery/blob/master/wikigraphvis/wikizeppelinnotebook.json

### The example is based on the databricks example from this link. 
https://docs.cloud.databricks.com/docs/latest/featured_notebooks/Wikipedia%20Clickstream%20Data.html

### The wikipedia web analytics data used is downloaded from the following website
https://datahub.io/dataset/wikipedia-clickstream

## Here are the steps used in the example

1. Add dependencies for databricks csv package. 
2. Download the wikipedia file and unzip it .
3. Read the file using spark dataframes
4. Write it to parquet format for fast reload on restarting Zeppelin. 
5. Load parquet file and register as a table
6. Summarise using Spark SQL
7. Create the reusable graph function 
8. Create the reusable displayForceLayout function to display in Force Layout format 
9. Create the reusable displaySankeyLayout
10. Run Example  on terms "Big_data" and "Data_science"

## 1. Add dependencies for databricks csv package. 
```
    %dep
    z.reset()

// Add spark-csv package
z.load("com.databricks:spark-csv_2.11:1.4.0") 
```

## 2. Download the wikipedia file and unzip it .
```
%sh mkdir wikipedia
cd wikipedia
wget https://ndownloader.figshare.com/files/3289898
mv 3289898 2015_02_clickstream.tsv.gz

gunzip  "2015_02_clickstream.tsv.gz"
```


## 3. Read the file using spark dataframes
```
val clickstreamDF = sqlContext.read.format("com.databricks.spark.csv")
  .option("header", "true") // first line is the header
  .option("delimiter", "\\t") // tab delimiter
  .option("mode", "PERMISSIVE")
  .option("inferSchema", "true") // infer data types (e.g., int, string) from    values
  .load("wikidata/2015_02_clickstream.tsv")
```


## 4. Write it to parquet format for fast reload on restarting Zeppelin.  
```
 clickstreamDF.write.parquet("wikipedia/wiki-clickstream-parquet")
```

## 5. Load parquet file and register as a table
```
val clicks = sqlContext.read.parquet("wikipedia/wiki-clickstream-parquet")
clicks.take(5)
clicks.registerTempTable("clicks")
clicks.printSchema
clicks.count()
```
## 6. Summarise using Spark SQL
```
%sql 
 SELECT 
      prev_title AS src,
      curr_title AS dest,
      n AS count FROM clicks
    WHERE 
     curr_title IN ('London', 'Dubai', 'Bangalore','New_York', 'San_Francisco') AND
      prev_id IS NOT NULL AND NOT (curr_title = 'Main_Page' OR prev_title = 'Main_Page')
    ORDER BY n DESC
    LIMIT 1000
```

## 7. Create the reusable graph function 
```
def generateGraphJson(query_terms :String, limitcnt :Int ): String = {
   
     
     val query_terms2 = query_terms.split(",")
   
    
     val sqlString =  """SELECT prev_title AS src, curr_title AS dest,n AS count FROM clicks 
    WHERE   curr_title IN ( """ +query_terms + """ ) AND 
      prev_id IS NOT NULL AND NOT (curr_title = 'Main_Page' OR prev_title = 'Main_Page') 
    ORDER BY n DESC LIMIT  """ +limitcnt    
    
    
    
    val clicks = sql( {sqlString}).as[Edge] 
 
  val data = clicks.collect()
  val nodes = (data.map(_.src) ++ data.map(_.dest)).map(_.replaceAll("_", " ")).toSet.toSeq.map(Node)
  val links = data.map { t =>
    Link(nodes.indexWhere(_.name == t.src.replaceAll("_", " ")), nodes.indexWhere(_.name == t.dest.replaceAll("_", " ")), t.count / 20 + 1)
  }
 
 
return  Seq(Graph(nodes, links)).toDF().toJSON.first() 
 
 
 
}

val cities = generateGraphJson("'London', 'Dubai', 'Bangalore','New_York', 'Tokyo', 'Sydney', 'Rio_de_Janeiro', 'Lagos' ", 40)
```


## 8. Create the reusable displayForceLayout function to display in Force Layout format 
```
  def displayForceLayout(data: String ,divname :String ) : Unit = {
     
     
     
 print(s"""%html

 <div id ="${divname}">  </div>
<style>

.node_circle {
  stroke: #777;
  stroke-width: 1.3px;
}

.node_label {
  pointer-events: none;
}

.link {
  stroke: #777;
  stroke-opacity: .2;
}

.node_count {
  stroke: #777;
  stroke-width: 1.0px;
  fill: #999;
}

text.legend {
  font-family: Verdana;
  font-size: 13px;
  fill: #000;
  color:#000;
}

.node text {
  font-family: "Helvetica Neue","Helvetica","Arial",sans-serif;
  font-size: 17px;
  font-weight: 200;
  color:#000;
}

</style>



<script src="http://d3js.org/d3.v3.min.js"></script>
<script>


var graph = JSON.parse( JSON.stringify( ${data}));

//var graph = JSON.parse( ${data});

var width = 800,
    height =600;

var color = d3.scale.category20();

var force = d3.layout.force()
    .charge(-700)
    .linkDistance(180)
    .size([width, height]);

var svg = d3.select("#${divname}").append("svg")
    .attr("width", width)
    .attr("height", height);
    
force
    .nodes(graph.nodes)
    .links(graph.links)
    .start();

var link = svg.selectAll(".link")
    .data(graph.links)
    .enter().append("line")
    .attr("class", "link")
    .style("stroke-width", function(d) { return Math.sqrt(d.value); });

var node = svg.selectAll(".node")
    .data(graph.nodes)
    .enter().append("g")
    .attr("class", "node")
    .call(force.drag);

node.append("circle")
    .attr("r", 10)
    .style("fill", function (d) {
    if (d.name.startsWith("other")) { return color(1); } else { return color(2); };
})

node.append("text")
      .attr("dx", 10)
      .attr("dy", ".35em")
      .text(function(d) { return d.name });
      
//Now we are giving the SVGs co-ordinates - the force layout is generating the co-ordinates which this code is using to update the attributes of the SVG elements
force.on("tick", function () {
    link.attr("x1", function (d) {
        return d.source.x;
    })
        .attr("y1", function (d) {
        return d.source.y;
    })
        .attr("x2", function (d) {
        return d.target.x;
    })
        .attr("y2", function (d) {
        return d.target.y;
    });
    d3.selectAll("circle").attr("cx", function (d) {
        return d.x;
    })
        .attr("cy", function (d) {
        return d.y;
    });
    d3.selectAll("text").attr("x", function (d) {
        return d.x;
    })
        .attr("y", function (d) {
        return d.y;
    });
});
 
</script>
 
""")
}

displayForceLayout(cities, "newdiv")
```

8. Create the reusable displayForceLayout function to display in Force Layout format 

## 4. Write it to parquet format for fast reload on restarting Zeppelin.  
```
 clickstreamDF.write.parquet("wikipedia/wiki-clickstream-parquet")
```


## 9. Create the reusable displaySankeyLayout
```
  def displaySankeyChart(data: String ,divname :String ) : Unit = {

 print(s"""%html

<div id ="${divname}" > </div>
<style>
.node rect {
  cursor: move;
  fill-opacity: .9;
  shape-rendering: crispEdges;
}
.node text {
  pointer-events: none;
  text-shadow: 0 1px 0 #fff;
}
.link {
  fill: none;
  stroke: #000;
  stroke-opacity: .2;
}
.link:hover {
  stroke-opacity: .5;
}
</style>
 
 
<script src="http://d3js.org/d3.v3.min.js"></script>
<script>
d3.sankey = function() {
  var sankey = {},
      nodeWidth = 24,
      nodePadding = 8,
      size = [1, 1],
      nodes = [],
      links = [];
  sankey.nodeWidth = function(_) {
    if (!arguments.length) return nodeWidth;
    nodeWidth = +_;
    return sankey;
  };
  sankey.nodePadding = function(_) {
    if (!arguments.length) return nodePadding;
    nodePadding = +_;
    return sankey;
  };
  sankey.nodes = function(_) {
    if (!arguments.length) return nodes;
    nodes = _;
    return sankey;
  };
  sankey.links = function(_) {
    if (!arguments.length) return links;
    links = _;
    return sankey;
  };
  sankey.size = function(_) {
    if (!arguments.length) return size;
    size = _;
    return sankey;
  };
  sankey.layout = function(iterations) {
    computeNodeLinks();
    computeNodeValues();
    computeNodeBreadths();
    computeNodeDepths(iterations);
    computeLinkDepths();
    return sankey;
  };
  sankey.relayout = function() {
    computeLinkDepths();
    return sankey;
  };
  sankey.link = function() {
    var curvature = .5;
    function link(d) {
      var x0 = d.source.x + d.source.dx,
          x1 = d.target.x,
          xi = d3.interpolateNumber(x0, x1),
          x2 = xi(curvature),
          x3 = xi(1 - curvature),
          y0 = d.source.y + d.sy + d.dy / 2,
          y1 = d.target.y + d.ty + d.dy / 2;
      return "M" + x0 + "," + y0
           + "C" + x2 + "," + y0
           + " " + x3 + "," + y1
           + " " + x1 + "," + y1;
    }
    link.curvature = function(_) {
      if (!arguments.length) return curvature;
      curvature = +_;
      return link;
    };
    return link;
  };
  // Populate the sourceLinks and targetLinks for each node.
  // Also, if the source and target are not objects, assume they are indices.
  function computeNodeLinks() {
    nodes.forEach(function(node) {
      node.sourceLinks = [];
      node.targetLinks = [];
    });
    links.forEach(function(link) {
      var source = link.source,
          target = link.target;
      if (typeof source === "number") source = link.source = nodes[link.source];
      if (typeof target === "number") target = link.target = nodes[link.target];
      source.sourceLinks.push(link);
      target.targetLinks.push(link);
    });
  }
  // Compute the value (size) of each node by summing the associated links.
  function computeNodeValues() {
    nodes.forEach(function(node) {
      node.value = Math.max(
        d3.sum(node.sourceLinks, value),
        d3.sum(node.targetLinks, value)
      );
    });
  }
  // Iteratively assign the breadth (x-position) for each node.
  // Nodes are assigned the maximum breadth of incoming neighbors plus one;
  // nodes with no incoming links are assigned breadth zero, while
  // nodes with no outgoing links are assigned the maximum breadth.
  function computeNodeBreadths() {
    var remainingNodes = nodes,
        nextNodes,
        x = 0;
    while (remainingNodes.length) {
      nextNodes = [];
      remainingNodes.forEach(function(node) {
        node.x = x;
        node.dx = nodeWidth;
        node.sourceLinks.forEach(function(link) {
          if (nextNodes.indexOf(link.target) < 0) {
            nextNodes.push(link.target);
          }
        });
      });
      remainingNodes = nextNodes;
      ++x;
    }
    //
    moveSinksRight(x);
    scaleNodeBreadths((size[0] - nodeWidth) / (x - 1));
  }
  function moveSourcesRight() {
    nodes.forEach(function(node) {
      if (!node.targetLinks.length) {
        node.x = d3.min(node.sourceLinks, function(d) { return d.target.x; }) - 1;
      }
    });
  }
  function moveSinksRight(x) {
    nodes.forEach(function(node) {
      if (!node.sourceLinks.length) {
        node.x = x - 1;
      }
    });
  }
  function scaleNodeBreadths(kx) {
    nodes.forEach(function(node) {
      node.x *= kx;
    });
  }
  function computeNodeDepths(iterations) {
    var nodesByBreadth = d3.nest()
        .key(function(d) { return d.x; })
        .sortKeys(d3.ascending)
        .entries(nodes)
        .map(function(d) { return d.values; });
    //
    initializeNodeDepth();
    resolveCollisions();
    for (var alpha = 1; iterations > 0; --iterations) {
      relaxRightToLeft(alpha *= .99);
      resolveCollisions();
      relaxLeftToRight(alpha);
      resolveCollisions();
    }
    function initializeNodeDepth() {
      var ky = d3.min(nodesByBreadth, function(nodes) {
        return (size[1] - (nodes.length - 1) * nodePadding) / d3.sum(nodes, value);
      });
      nodesByBreadth.forEach(function(nodes) {
        nodes.forEach(function(node, i) {
          node.y = i;
          node.dy = node.value * ky;
        });
      });
      links.forEach(function(link) {
        link.dy = link.value * ky;
      });
    }
    function relaxLeftToRight(alpha) {
      nodesByBreadth.forEach(function(nodes, breadth) {
        nodes.forEach(function(node) {
          if (node.targetLinks.length) {
            var y = d3.sum(node.targetLinks, weightedSource) / d3.sum(node.targetLinks, value);
            node.y += (y - center(node)) * alpha;
          }
        });
      });
      function weightedSource(link) {
        return center(link.source) * link.value;
      }
    }
    function relaxRightToLeft(alpha) {
      nodesByBreadth.slice().reverse().forEach(function(nodes) {
        nodes.forEach(function(node) {
          if (node.sourceLinks.length) {
            var y = d3.sum(node.sourceLinks, weightedTarget) / d3.sum(node.sourceLinks, value);
            node.y += (y - center(node)) * alpha;
          }
        });
      });
      function weightedTarget(link) {
        return center(link.target) * link.value;
      }
    }
    function resolveCollisions() {
      nodesByBreadth.forEach(function(nodes) {
        var node,
            dy,
            y0 = 0,
            n = nodes.length,
            i;
        // Push any overlapping nodes down.
        nodes.sort(ascendingDepth);
        for (i = 0; i < n; ++i) {
          node = nodes[i];
          dy = y0 - node.y;
          if (dy > 0) node.y += dy;
          y0 = node.y + node.dy + nodePadding;
        }
        // If the bottommost node goes outside the bounds, push it back up.
        dy = y0 - nodePadding - size[1];
        if (dy > 0) {
          y0 = node.y -= dy;
          // Push any overlapping nodes back up.
          for (i = n - 2; i >= 0; --i) {
            node = nodes[i];
            dy = node.y + node.dy + nodePadding - y0;
            if (dy > 0) node.y -= dy;
            y0 = node.y;
          }
        }
      });
    }
    function ascendingDepth(a, b) {
      return a.y - b.y;
    }
  }
  function computeLinkDepths() {
    nodes.forEach(function(node) {
      node.sourceLinks.sort(ascendingTargetDepth);
      node.targetLinks.sort(ascendingSourceDepth);
    });
    nodes.forEach(function(node) {
      var sy = 0, ty = 0;
      node.sourceLinks.forEach(function(link) {
        link.sy = sy;
        sy += link.dy;
      });
      node.targetLinks.forEach(function(link) {
        link.ty = ty;
        ty += link.dy;
      });
    });
    function ascendingSourceDepth(a, b) {
      return a.source.y - b.source.y;
    }
    function ascendingTargetDepth(a, b) {
      return a.target.y - b.target.y;
    }
  }
  function center(node) {
    return node.y + node.dy / 2;
  }
  function value(link) {
    return link.value;
  }
  return sankey;
};
</script>
<script>
	
var margin = {top: 1, right: 1, bottom: 6, left: 1},
    width = 960 - margin.left - margin.right,
    height = 500 - margin.top - margin.bottom;

var formatNumber = d3.format(",.0f"),
    format = function(d) { return formatNumber(d) + " TWh"; },
    color = d3.scale.category20();

var svg = d3.select("#${divname}").append("svg")
    .attr("width", width + margin.left + margin.right)
    .attr("height", height + margin.top + margin.bottom)
  .append("g")
    .attr("transform", "translate(" + margin.left + "," + margin.top + ")");

var sankey = d3.sankey()
    .nodeWidth(15)
    .nodePadding(10)
    .size([width, height]);

var path = sankey.link();

var graph = JSON.parse( JSON.stringify( ${ data}));
sankey
      .nodes(graph.nodes)
      .links(graph.links)
      .layout(32);

  var link = svg.append("g").selectAll(".link")
      .data(graph.links)
    .enter().append("path")
      .attr("class", "link")
      .attr("d", path)
      .style("stroke-width", function(d) { return Math.max(1, d.dy); })
      .sort(function(a, b) { return b.dy - a.dy; });

  link.append("title")
      .text(function(d) { return d.source.name + " → " + d.target.name + "test" + format(d.value); });

  var node = svg.append("g").selectAll(".node")
      .data(graph.nodes)
    .enter().append("g")
      .attr("class", "node")
      .attr("transform", function(d) { return "translate(" + d.x + "," + d.y + ")"; })
    .call(d3.behavior.drag()
      .origin(function(d) { return d; })
      .on("dragstart", function() { this.parentNode.appendChild(this); })
      .on("drag", dragmove));

  node.append("rect")
      .attr("height", function(d) { return d.dy; })
      .attr("width", sankey.nodeWidth())
      .style("fill", function(d) { return d.color = color(d.name.replace(/ .*/, "")); })
      .style("stroke", function(d) { return d3.rgb(d.color).darker(2); })
    .append("title")
      .text(function(d) { return d.name + "test" + format(d.value); });

  node.append("text")
      .attr("x", -6)
      .attr("y", function(d) { return d.dy / 2; })
      .attr("dy", ".35em")
      .attr("text-anchor", "end")
      .attr("transform", null)
      .text(function(d) { return d.name; })
    .filter(function(d) { return d.x < width / 2; })
      .attr("x", 6 + sankey.nodeWidth())
      .attr("text-anchor", "start");

  function dragmove(d) {
    d3.select(this).attr("transform", "translate(" + d.x + "," + (d.y = Math.max(0, Math.min(height - d.dy, d3.event.y))) + ")");
    sankey.relayout();
    link.attr("d", path);
  }
 

 
 
</script>



""");
}
 
displaySankeyChart(cities,"sankeydiv1")
```



## 10. Run Force Layout Example  on pages on "Big_data" and "Data_science"  
```
 val tech   = generateGraphJson("    'Big_data', 'Data_science' ", 20)

displayForceLayout(tech,"div2")
``` 
## 11. Run Sankey Example  on pages on "Big_data" and "Data_science"  
```
 
displaySankeyChart(tech,"bigdata")
```

## 12. Force Layout Output Example ( Github html preview output is duplicating the chart. Only the second chart is displayed when opening html directly in a browser)

http://htmlpreview.github.io/?https://github.com/deepakas/zeppelin-gallery/blob/master/wikigraphvis/d3forcelayoutex.html

## 13. Sankey Layout Output Example ( Github html preview output is duplicating the chart. Only the second chart is displayed when opening html directly in a browser)

http://htmlpreview.github.io/?https://github.com/deepakas/zeppelin-gallery/blob/master/wikigraphvis/d3sankeylayoutex.html



