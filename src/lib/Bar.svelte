<script lang="ts">
  import type { TMovie } from "../types";
  import * as d3 from "d3";
  // define the props of the Bar component
  type Props = {
    movies: TMovie[];
    progress?: number;
    width?: number;
    height?: number;
  };
  // progress is 100 by default unless specified
  let { movies, progress = 100, width = 500, height = 400 }: Props = $props();

  let selectedGenre: string = $state(''); //give starting value 

  // processing the data; $derived is used to create a reactive variable that updates whenever the dependent variables change
  const yearRange = $derived(d3.extent(movies.map((d) => d.year)));

  function getUpYear(yearRange: [undefined, undefined] | [Date, Date]) {
    if (!yearRange[0]) return new Date();
    const timeScale = d3.scaleTime().domain(yearRange).range([0, 100]);
    return timeScale.invert(progress);
  }
  const upYear: Date = $derived(getUpYear(yearRange!));

  function getGenreNums(movies: TMovie[], upYear: Date) {
    let res: { [genre: string]: number } = {};
    movies
      .filter((movie) => movie.year <= upYear)
      .forEach((movie) => {
        movie.genres.forEach((genre: string) => {
          res[genre] = (res[genre] || 0) + 1;
        });
      });
    return res;
  }

  const genreNums = $derived(getGenreNums(movies, upYear));
  //const genreNumsFull = $derived(getGenreNums(movies, new Date(Math.max(...movies.map((movie) => movie.year.getTime())))));
  //new Date(Math.max(...dates));
  const genreNumsFull = $derived(getGenreNums(movies, new Date(2023,0,1)));

  // drawing the bar chart

  const margin = {
    top: 15,
    bottom: 50,
    left: 30,
    right: 10,
  };

  let usableArea = {
    top: margin.top,
    right: width - margin.right,
    bottom: height - margin.bottom,
    left: margin.left,
  };

  const xScale = $derived(
    // tip: use d3.scaleBand() to create a band scale for the x-axis
    d3
      .scaleBand()
      .range([usableArea.left, usableArea.right]) //y axis array of 2 numbers
      .domain(Object.keys(genreNumsFull)) //x axis
      .padding(.09),
  );

  const yScale = $derived(
    // tip: use d3.scaleLinear() to create a linear scale for the y-axis
    d3
      .scaleLinear()
      .range([usableArea.bottom, usableArea.top])
      .domain([0, d3.max(Object.values(genreNumsFull)) ?? 0])
  );
  // const yScaleFull = $derived(
  //   // tip: use d3.scaleLinear() to create a linear scale for the y-axis
  //   d3
  //     .scaleLinear()
  //     .range([usableArea.bottom, usableArea.top])
  //     .domain([0, d3.max(Object.values(genreNumsFull)) ?? 0])
  // );

  const xBarwidth: number = $derived(xScale.bandwidth());

  let xAxis: any = $state(),
    yAxis: any = $state();

  function updateAxis() {
    d3.select(xAxis)
      .call(d3.axisBottom(xScale))
      .selectAll("text")
      .attr("transform", "rotate(45)")
      .style("text-anchor", "start");

    // tip:
    // similar to the x-axis, create a y-axis using d3.axisLeft() and bind it to the yAxis variable
    d3.select(yAxis)
      .call(d3.axisLeft(yScale))
      .selectAll("text")
      //.attr("transform", "rotate(45)")
      .style("text-anchor", "end");
  }

  // the $effect function is used to run a function whenever the reactive variables change, also known as a side effect
  $effect(() => {
    //console.log(genreNums)
    console.log(genreNumsFull);
    console.log(xScale)
    updateAxis();
  });
  
</script>

<h3>
  The Distribution of Genres {yearRange[0]?.getFullYear()} - {upYear.getFullYear()}
</h3>

{#if movies.length > 0}
  <svg {width} {height}>
    <g class="bars">
      {#each Object.keys(genreNumsFull) as genre}
        <g class={genre}>
          <!-- tip: draw bars here using the svg <rect/> component -->
          <rect
            width={xBarwidth}
            height={yScale(0) - yScale(genreNums[genre])}
            x={xScale(genre)}
            y={yScale(genreNums[genre])}
            fill="#449900"
            class="bar"
            opacity={selectedGenre == genre ? 1.0 : 0.5}
            onmouseover={() => {
              selectedGenre = genre;
            }}
            onmouseout={() => {
              selectedGenre = ''
            }}
          />

          <rect
            width={xBarwidth}
            height={yScale(0) - yScale(genreNumsFull[genre])}
            x={xScale(genre)}
            y={yScale(genreNumsFull[genre])}
            fill="#449900"
            class="bar"
            opacity={selectedGenre == genre ? 1.0 : 0.5}
            onmouseover={() => {
              selectedGenre = genre;
            }}
            onmouseout={() => {
              selectedGenre = ''
            }}
          />


          <text
            x={xScale(genre)! + xBarwidth / 2}
            y={yScale(genreNumsFull[genre]) - 5}
            font-size="12"
            text-anchor="middle"
          >
          <!-- tip: the text below should change with the hover on interaction -->
          {#if selectedGenre == genre}
            {genre}: {genreNumsFull[genre]} 
          {:else}
            {genreNumsFull[genre]} 
          {/if}
          </text>
          <text
            x={xScale(genre)! + xBarwidth / 2}
            y={yScale(genreNums[genre]) - 5}
            font-size="12"
            text-anchor="middle"
          >
          <!-- tip: the text below should change with the hover on interaction -->
          {#if selectedGenre == genre}
            {genre}: {genreNums[genre]} 
          {:else}
            {genreNums[genre]} 
          {/if}
          </text>
        </g>
      {/each}
    </g>

    <g transform="translate(0, {usableArea.bottom})" bind:this={xAxis} />
    <g transform="translate({usableArea.left}, 0)" bind:this={yAxis} />
  </svg>
{/if}

<style>
  .bar {
    transition:
      y 0.1s ease,
      height 0.1s ease,
      width 0.1s ease; /* Smooth transition for height */
  }
</style>