<script lang="ts">
  import { onMount } from "svelte";
  import * as d3 from "d3";
  import type { TMovie } from "../../types";
  import StoryOpen from "./StoryOpen.svelte";
  import Scrolly2D from "./Scrolly2D.svelte";
  import Scrolly3D from "./Scrolly3D.svelte";


  let derivativeData: any[] = []; // The delta values for your graph
  let genreList: string[] = [];

  let movies: TMovie[] = [];

  // Reactive variable for storing the data

  function initYearEntry(year: number): any {
    let entry: any = {};
    genreList.forEach((genre) => {
      entry[genre] = 0;
    });
    return entry;
  }

  // Function to load the CSV
  async function loadCsv() {
    try {
      const csvUrl = "./summer_movies.csv";
      movies = await d3.csv(csvUrl, (row) => {
        // all values are strings, so use row conversion function to format them
        return {
          ...row,
          num_votes: Number(row.num_votes),
          runtime_minutes: Number(row.runtime_minutes),
          genres: row.genres.split(","),
          year: new Date(row.year),
          average_rating: Number(row.average_rating),
        };
      });
      let minYear = new Date().getFullYear();
      let maxYear = 1900;
      for (let i = 0; i < movies.length; i++) {
        let curGenres = movies[i].genres;
        for (let j = 0; j < curGenres.length; j++) {
          let aGenre = curGenres[j];
          if (!genreList.includes(aGenre)) {
            genreList.push(aGenre);
          }
        }

        let thisYear = movies[i].year.getFullYear();
        if (thisYear < minYear) {
          minYear = thisYear;
        }
        if (thisYear > maxYear) {
          maxYear = thisYear;
        }
      }
      console.log("Unique Genres:", genreList);

      interface YearGenreCount {
        [genre: string]: number;
      }
      interface StackData {
        [key: string]: YearGenreCount;
      }
      let stackData: StackData = {}; // The raw year-by-year counts
      for (let j = 0; j < movies.length; j++) {
        let thisYear = movies[j].year.getFullYear();

        // Check if this year is already in stackData
        let yearEntry = stackData[thisYear];
        if (!yearEntry) {
          yearEntry = initYearEntry(thisYear);
        }

        // Increment the count for each genre in this movie
        for (let k = 0; k < movies[j].genres.length; k++) {
          let aGenre = movies[j].genres[k];
          yearEntry[aGenre] += 1;
        }

        stackData[thisYear] = yearEntry;
      }

      // Calculate change from year to year
      for (let year = minYear; year < maxYear - 1; year++) {
        const current = stackData[year] || initYearEntry(year); // Default to zero counts if year is missing
        const previous = stackData[year - 1] || initYearEntry(year - 1);

        let yearDiff: any = { year: year };
        genreList.forEach((genre) => {
          // Derivative = (Current Year Count) - (Previous Year Count)
          yearDiff[genre] = (current[genre] || 0) - (previous[genre] || 0);
        });
        derivativeData.push(yearDiff);
      }
      derivativeData = derivativeData;
      console.log("Loaded CSV Data:", movies);
    } catch (error) {
      console.error("Error loading CSV:", error);
    }

    console.log(derivativeData);
  }
  // Call the loader when the component mounts
  onMount(loadCsv);
</script>

<div class="container">
  {#if derivativeData.length > 0}
    <StoryOpen movieNum={movies.length} />
    <Scrolly2D {movies} />
    <Scrolly3D {derivativeData} {genreList} />
  {/if}
</div>

<style>
  .container {
    width: 80vw;
    margin: 10px auto;
    padding: 10px;
    align-content: center;
  }
</style>
