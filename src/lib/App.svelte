<script>
    import { onMount } from 'svelte';
    import * as d3 from 'd3';

    let tempData = [];
    let root = null;
    let height = 0;

    onMount(async () => {
        const res = await fetch('cleaned_yelp_dataset_business.csv'); 
        const csv = await res.text();
        tempData = d3.csvParse(csv, d3.autoType)
        // console.log(tempData);
    
        // CODE TO FORMAT DATA BELOW
        let nestedData = {};

        tempData.forEach(d => {
            if (!nestedData[d.state]) {
                nestedData[d.state] = {
                    totalBusinesses: 0,
                    categoryCounts: {}
                };
            }

            nestedData[d.state].totalBusinesses++;

            if (!nestedData[d.state].categoryCounts[d.category]) {
                nestedData[d.state].categoryCounts[d.category] = {
                    totalBusinesses: 0,
                    starCounts: {}
                };
            }

            nestedData[d.state].categoryCounts[d.category].totalBusinesses++;

            if (!nestedData[d.state].categoryCounts[d.category].starCounts[d.stars]) {
                nestedData[d.state].categoryCounts[d.category].starCounts[d.stars] = 0;
            }

            nestedData[d.state].categoryCounts[d.category].starCounts[d.stars]++;
        });

        const dataAsArray = Object.entries(nestedData).map(([name, children]) => ({
            name,
            children: Object.entries(children.categoryCounts).map(([category, { totalBusinesses, starCounts }]) => ({
                name: category,
                children: Object.entries(starCounts).map(([star, count]) => ({ name: star.toString(), value: count })),
                value: totalBusinesses
            }))
        }));

        const rootData = { name: "data", children: dataAsArray };

        root = d3.hierarchy(rootData)
            .sum(d => d.value)
            .sort((a, b) => b.value - a.value)
            .eachAfter(d => d.index = d.parent ? d.parent.index = d.parent.index + 1 || 0 : 0);
        
        // console.log(root);
    });

    // DECLARE VARIABLES BELOW
    $: if (root !== null) {
        height = calc_height();
        chart();
    }

    let width = 1250; // CHANGE WIDTH HERE
    let barStep = 27;
    let barPadding = 3 / barStep;
    let duration = 750;
    let marginTop = 30;
    let marginRight = 30;
    let marginBottom = 0;
    let marginLeft = 100;
    

    let x = d3.scaleLinear().range([marginLeft, width - marginRight])

    let xAxis = g => g
        .attr("class", "x-axis")
        .attr("transform", `translate(0,${marginTop})`)
        .call(d3.axisTop(x).ticks(width / 80, "s"))
        .call(g => (g.selection ? g.selection() : g).select(".domain").remove())

    let yAxis = g => g
        .attr("class", "y-axis")
        .attr("transform", `translate(${marginLeft + 0.5},0)`)
        .call(g => g.append("line")
            .attr("stroke", "currentColor")
            .attr("y1", marginTop)
            .attr("y2", height - marginBottom))

    let color = d3.scaleOrdinal().range(["steelblue"]);


    // ALL FUNCTIONS BELOW
    function calc_height() {
        let max = 1;
        root.each(d => d.children && (max = Math.max(max, d.children.length)));
        return max * barStep + marginTop + marginBottom;
    }

    function chart() {
        let svg = d3.create("svg")
            .attr("viewBox", [0, 0, width, height])
            .attr("width", width)
            .attr("height", height + 25)
            .attr("style", "max-width: 100%; height: auto;")
            .attr("transform", "translate(" + 35 + ", 0)");

        x.domain([0, root.value]);

        svg.append("rect")
            .attr("class", "background")
            .attr("fill", "none")
            .attr("pointer-events", "all")
            .attr("width", width)
            .attr("height", height)
            .attr("cursor", "pointer")
            .on("click", (event, d) => up(svg, d));

        svg.append("g")
            .call(xAxis);

        svg.append("g")
            .call(yAxis);
        
        // For x-axis label
        svg.append("text")
            .attr("class", "x-axis-label")
            .attr("text-anchor", "middle")
            .attr("x", width / 2)
            .attr("y", 0)
            .attr("font-weight", "bold")
            .text("Count");

        // Add y-axis label
        svg.append("text")
            .attr("class", "y-axis-label")
            .attr("text-anchor", "middle")
            .attr("transform", "rotate(-90)")
            .attr("x", -height / 2)
            .attr("y", 12)
            .attr("font-weight", "bold")
            .text("Businesses Across U.S. States");

        down(svg, root);

        const container = document.getElementById('chart-container');
        if (container) { // Check if the container exists before appending
            container.innerHTML = ''; // Clear existing content
            container.appendChild(svg.node());
        }
    }

    // Creates a set of bars for the given data node, at the specified index.
    // Inside the `bar` function
    function bar(svg, down, d, selector) {
        const g = svg.insert("g", selector)
            .attr("class", "enter")
            .attr("transform", `translate(0,${marginTop + barStep * barPadding})`)
            .attr("text-anchor", "end")
            .style("font", "10px sans-serif");

        const bar = g.selectAll("g")
            .data(d.children)
            .join("g")
            .attr("cursor", d => !d.children ? null : "pointer")
            .on("click", (event, d) => down(svg, d));

        bar.append("text")
            .attr("x", marginLeft - 6)
            .attr("y", barStep * (1 - barPadding) / 2)
            .attr("dy", ".35em")
            .text(d => d.data.name);

        bar.append("rect")
            .attr("x", x(0))
            .attr("width", d => x(d.value) - x(0))
            .attr("height", barStep * (1 - barPadding))
            .attr("fill", d => color(!!d.children)) // Set initial bar color
            .on("mouseover", function () {
                d3.select(this)
                    .transition()
                    .duration(500) // Smooth transition duration
                    .attr("fill", "#af69ee"); // purple when hover

                d3.select(this.parentNode).select("text")
                    .transition()
                    .duration(500) // Smooth transition duration
                    .attr("fill", "#af69ee"); // purple when hover
            })
            .on("mouseout", function (event, d) {
                d3.select(this)
                    .transition()
                    .duration(500) // Smooth transition duration
                    .attr("fill", color(!!d.children)); // Restore the original bar color on mouseout

                d3.select(this.parentNode).select("text")
                    .transition()
                    .duration(500) // Smooth transition duration
                    .attr("fill", "black"); // Restore the original text color on mouseout
            });

        // Append text for counts
        bar.append("text")
            .attr("class", "bar-count")
            .attr("x", d => x(d.value)) // Adjust position for better alignment
            .attr("y", barStep * (1 - barPadding) / 2)
            .attr("dx", 3)
            .attr("dy", ".35em")
            .attr("fill", "white")
            .attr("text-anchor", "start")
            .text(d => d.value); // Display count value

        return g;
    }




    function down(svg, d) {
        if (!d.children || d3.active(svg.node())) return;

        // Rebind the current node to the background.
        svg.select(".background").datum(d);

        // Define two sequenced transitions.
        const transition1 = svg.transition().duration(duration);
        const transition2 = transition1.transition();

        // Mark any currently-displayed bars as exiting.
        const exit = svg.selectAll(".enter")
            .attr("class", "exit");

        // Entering nodes immediately obscure the clicked-on bar, so hide it.
        exit.selectAll("rect")
            .attr("fill-opacity", p => p === d ? 0 : null);

        // Transition exiting bars to fade out.
        exit.transition(transition1)
            .attr("fill-opacity", 0)
            .remove();

        // Enter the new bars for the clicked-on data.
        // Per above, entering bars are immediately visible.
        const enter = bar(svg, down, d, ".y-axis")
            .attr("fill-opacity", 0);

        // Have the text fade-in, even though the bars are visible.
        enter.transition(transition1)
            .attr("fill-opacity", 1);

        // Transition entering bars to their new y-position.
        enter.selectAll("g")
            .attr("transform", stack(d.index))
            .transition(transition1)
            .attr("transform", stagger());

        // Update the x-scale domain.
        x.domain([0, d3.max(d.children, d => d.value)]);

        // Update the x-axis.
        svg.selectAll(".x-axis").transition(transition2)
            .call(xAxis);

        // Transition entering bars to the new x-scale.
        enter.selectAll("g").transition(transition2)
            .attr("transform", (d, i) => `translate(0,${barStep * i})`);

        // Color the bars as parents; they will fade to children if appropriate.
        enter.selectAll("rect")
            .attr("fill", color(true))
            .attr("fill-opacity", 1)
            .transition(transition2)
            .attr("fill", d => color(!!d.children))
            .attr("width", d => x(d.value) - x(0));


        // Update the y-axis label text
        let yAxisLabel = '';
        if (d.depth === 0) {
            yAxisLabel = 'Businesses Across US States';
        } else if (d.depth == 1){
            yAxisLabel = 'Distribution of Businesses in ' + d.data.name;
        } else {
            yAxisLabel = 'Star Distribution of ' + d.data.name + ' in ' + d.parent.data.name;
        }
        svg.selectAll(".y-axis-label")
            .text(yAxisLabel); // Set label text to the current category
    }

    function up(svg, d) {
        if (!d.parent || !svg.selectAll(".exit").empty()) return;

        // Rebind the current node to the background.
        svg.select(".background").datum(d.parent);

        // Define two sequenced transitions.
        const transition1 = svg.transition().duration(duration);
        const transition2 = transition1.transition();

        // Mark any currently-displayed bars as exiting.
        const exit = svg.selectAll(".enter")
            .attr("class", "exit");

        // Update the x-scale domain.
        x.domain([0, d3.max(d.parent.children, d => d.value)]);

        // Update the x-axis.
        svg.selectAll(".x-axis").transition(transition1)
            .call(xAxis);

        // Transition exiting bars to the new x-scale.
        exit.selectAll("g").transition(transition1)
            .attr("transform", stagger());

        // Transition exiting bars to the parent’s position.
        exit.selectAll("g").transition(transition2)
            .attr("transform", stack(d.index));

        // Transition exiting rects to the new scale and fade to parent color.
        exit.selectAll("rect").transition(transition1)
            .attr("width", d => x(d.value) - x(0))
            .attr("fill", color(true));

        // Transition exiting text to fade out.
        // Remove exiting nodes.
        exit.transition(transition2)
            .attr("fill-opacity", 0)
            .remove();

        // Enter the new bars for the clicked-on data's parent.
        const enter = bar(svg, down, d.parent, ".exit")
            .attr("fill-opacity", 0);

        enter.selectAll("g")
            .attr("transform", (d, i) => `translate(0,${barStep * i})`);

        // Transition entering bars to fade in over the full duration.
        enter.transition(transition2)
            .attr("fill-opacity", 1);

        // Color the bars as appropriate.
        // Exiting nodes will obscure the parent bar, so hide it.
        // Transition entering rects to the new x-scale.
        // When the entering parent rect is done, make it visible!
        enter.selectAll("rect")
            .attr("fill", d => color(!!d.children))
            .attr("fill-opacity", p => p === d ? 0 : null)
            .transition(transition2)
            .attr("width", d => x(d.value) - x(0))
            .on("end", function(p) { d3.select(this).attr("fill-opacity", 1); });


        // Update the y-axis label text
        let yAxisLabel = '';
        if (d.parent.depth === 0) {
            yAxisLabel = 'Businesses Across US States';
        } else if (d.parent.depth == 1){
            yAxisLabel =  'Distribution of Businesses in ' + d.parent.data.name;
        } else {
            yAxisLabel = 'Star Distribution of ' + d.parent.data.name + ' in ' + d.parent.parent.data.name;
        }
        svg.selectAll(".y-axis-label")
            .text(yAxisLabel); // Set label text to the current category
    }

    function stack(i) {
        let value = 0;
        return d => {
            const t = `translate(${x(value) - x(0)},${barStep * i})`;
            value += d.value;
            return t;
        };
    }

    function stagger() {
        let value = 0;
        return (d, i) => {
            const t = `translate(${x(value) - x(0)},${barStep * i})`;
            value += d.value;
            return t;
        };
    }
</script>

<h1>What Lies Behind Yelp Across America? Exploring Business Distribution, Types, and Ratings Across States</h1>
<p>Click a blue bar to drill down, or click the background to go back up.</p>

<main id="chart-container">
    {#if height > 0}
        <svg id="chart-svg" viewBox="0 0 {width} {height}"></svg>
    {/if}
</main>

<style>
    /* Title style */
    h1 {
        font-size: 2.5rem;
        margin-top: 50px;
        color: #007bff; /* Blue color */
        font-family: Arial, sans-serif; /* Specify a fallback font family */
        text-align: center; /* Center align the title */
    }

    /* Description style */
    p {
        font-size: 1.2rem;
        margin-bottom: 50px;
        color: #333; /* Darker gray color for better readability */
        font-family: Arial, sans-serif; /* Specify a fallback font family */
        text-align: center; /* Center align the description */
    }

    

</style>

