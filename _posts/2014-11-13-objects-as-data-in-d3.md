---
layout: post
category: blog
tags: [d3js, data-binding]
date: 2014-11-13 11:18:00 -0500
---
{% include JB/setup %}

[D3](http://d3js.org/) uses a very powerful data binding approach. However, many D3 examples show how to deal with very simple data, such as `[1,2,3]`. Let's have a look at what happens when data items are objects, for example

{% highlight javascript %}
var data = [
  {x: 70, y: 20, label: "first"},
  {x: 10, y: 60, label: "second"}
];
{% endhighlight %}

<!-- more -->

The binding itself does not change when going from numbers to objects - just bind data to your selection as before and go ahead with the usual D3 functions-as-values approach

{% highlight javascript %}
selection
	.data(data)
	.attr("x", function(d) { return d.x; });
{% endhighlight %}

Inspecting a DOM element from the selection and navigating to its properties shows that the `__data__` property is set to the corresponding item from the list (this is how d3 data binding internally works):

<img src="/files/2014-11-13-objects-as-data-in-d3/data_devtools.png" class="imgmb" alt="Data in developer tools"/>

Now let's visualize our data as `SVG Text` objects with text coming from `d.label` positioned at `(d.x, d.y)`, and add a couple of transitions:

{% highlight javascript %}
function render() {
    var svg = d3.select("#viz svg");
    var ditems = svg.selectAll("text").data(data);

    // enter
    ditems.enter()
        .append("text");

    // update
    ditems.transition().duration(500)
        .attr("x", function(d) { return d.x; })
        .attr("y", function(d) { return d.y; })
        .text(function(d) { return d.label; });

    // exit
    ditems.exit()
        .transition()
        .style("opacity", 0)
        .remove();
}
{% endhighlight %}

Here, the `update` transition will interpolate object coordinates, and the `exit` transition makes the text fade out before it is removed from the DOM. Here is what it looks like:

<style type="text/css">
.d3ex {
    margin-left :30px;
    margin-bottom: 20px;
    margin-top: 20px;
}
.svgExample {
    background-color: #eee;
    width: 130px;
    height: 80px;
}
#viz {
	display: inline-block;
	vertical-align: top;
}
.rblock {
	display: inline-block;
	vertical-align: top;
	margin-left: 10px;
}
.dataPreview {
	height: 45px;
	padding: 2px;
	font-family: Menlo, Monaco, Consolas, 'Courier New', monospace;
	font-size: 13px;
}
</style>

<div id="ex1" class="d3ex">
    <div id="viz"></div>
    <div class="rblock">
    	<div class="dataPreview">text</div>
    	<div>
		    <button type="button" class="btn btn-sm btn-default" id="bmove">Move</button>
		    <button type="button" class="btn btn-sm btn-default" id="brm">Remove</button>
		    <button type="button" class="btn btn-sm btn-default" id="breset">Reset</button>
		</div>
	</div>
</div>

But what happens if we remove the first object? Give it a try:

<div id="ex2" class="d3ex">
    <div id="viz"></div>
    <div class="rblock">
    	<div class="dataPreview">text</div>
    	<div>
		    <button type="button" class="btn btn-sm btn-default" id="brm">Remove first</button>
		    <button type="button" class="btn btn-sm btn-default" id="breset">Reset</button>
		</div>
	</div>
</div>

That's weird. How about swapping the objects in the `data` array without changing their contents?

<div id="ex3" class="d3ex">
    <div id="viz"></div>
    <div class="rblock">
    	<div class="dataPreview">text</div>
    	<div>
		    <button type="button" class="btn btn-sm btn-default" id="bswap">Swap</button>
		</div>
	</div>
</div>

As we only re-ordered the objects and still see some changes, it has something to do with indexes. In fact, by default D3 assumes that the objects are identified by their index in the data array, and this works well in many scenarios. Here, however, we don't want to use indexes and need to identify objects differently, which can be done by passing a `key` function to [selection.data(values, key)](https://github.com/mbostock/d3/wiki/Selections#data). For example, if we know the `label` is going to be unique then we can simply make the key function return it:

{% highlight javascript %}
selection.data(data, function(d) { return d.label; })
{% endhighlight %}

With that, swapping the objects will now have no effect and removing the first element will behave as expected:

<div id="ex4" class="d3ex">
    <div id="viz"></div>
    <div class="rblock">
    	<div class="dataPreview">text</div>
    	<div>
		    <button type="button" class="btn btn-sm btn-default" id="bswap">Swap</button>
		    <button type="button" class="btn btn-sm btn-default" id="brm">Remove first</button>
		    <button type="button" class="btn btn-sm btn-default" id="breset">Reset</button>
		</div>
	</div>
</div>

In case when labels are not necessarily unique, we would need to use something else as an identifier. For example, an `id` field could be added to each object and used as the key.

Happy data-binding!

<script src="http://ajax.googleapis.com/ajax/libs/jquery/1.10.2/jquery.min.js"></script>
<script src="http://cdnjs.cloudflare.com/ajax/libs/d3/3.4.13/d3.min.js"></script>

<script type="text/javascript">
function d3ex(id, dataId) {
	dataId = dataId || function(d, i) { return i; }

	var data0 = [{x: 70, y:20, label: "first"}, {x: 10, y:60, label: "second"}];
	var data;

	function init() {
	    data = JSON.parse(JSON.stringify(data0));

	    var svg = d3.select("#" + id + " #viz")
	        .html("<svg class='svgExample'></svg>");
	}

    function render() {
        var svg = d3.select("#" + id + " #viz svg");
        var ditems = svg.selectAll("text").data(data, dataId);

        // enter
        ditems.enter()
            .append("text");

        // update
        ditems.transition().duration(500)
            .attr("x", function(d) { return d.x; })
            .attr("y", function(d) { return d.y; })
            .text(function(d) { return d.label; });

        // exit
        ditems.exit()
            .transition()
            .style("opacity", 0)
            .remove();

        updPreview();
    }

    function reset() {
	    init();
	    render();
	}

	function removeFirst() {
		if (data.length != 2)
			return;
		if (data[0].label == "first")
			data.shift();
		else
			data.pop();
		render();
	}

	function removeLast() {
		data.pop();
		render();
	}

	function exMove() {
		for (var i=0; i<data.length; i++) {
			data[i].x = 10 + Math.floor(Math.random() * 60);
			data[i].y = Math.floor(15 + i* 30 + Math.random() * 15);
		}
		render();
	}

	function exSwap() {
		data.reverse();
		render();
	}

	function updPreview() {
		$("#" + id + " .dataPreview").html(JSON.stringify(data).replace("},", "},<br>&nbsp;"));
	}

	reset();

	return {
		reset: reset,
		render: render,
		removeFirst: removeFirst,
		exMove: exMove,
		removeLast: removeLast,
		exSwap: exSwap
	};
}
</script>

<script type="text/javascript">
$(document).ready(function() {
    var ex1 = d3ex("ex1");
    $("#ex1 #bmove").click(ex1.exMove);
    $("#ex1 #brm").click(ex1.removeLast);
    $("#ex1 #breset").click(ex1.reset);

    var ex2 = d3ex("ex2");
    $("#ex2 #brm").click(ex2.removeFirst);
    $("#ex2 #breset").click(ex2.reset);

    var ex3 = d3ex("ex3");
    $("#ex3 #bswap").click(ex3.exSwap);

    var ex4 = d3ex("ex4", function(d, i) { return d.label; });
    $("#ex4 #bswap").click(ex4.exSwap);
    $("#ex4 #brm").click(ex4.removeFirst);
    $("#ex4 #breset").click(ex4.reset);
});
</script>