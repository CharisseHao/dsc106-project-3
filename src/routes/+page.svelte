<main>
    <section>
        <h1>
        Yelp Across America: Analyzing Business Distribution, Types, and Ratings Across States
        </h1>
        <p> Click a blue bar to drill down, or click the background to go back up. </p>
    </section>
</main>

<script>
    import App from '../lib/App.svelte';
</script>

<App />

<style>
    h1 {
        font-family: "Times New Roman"; 
        font-size: 24px;
        color: #000;
    }

    section {
        font-family: "Times New Roman"; 
        font-size: 16px; 
        color: #333;
        text-align: center;
    }

    .chart-container {
        display: inline-block;
        border: 2px solid #333; /* Add border around the chart */
        padding: 10px; /* Add padding around the chart */
    }
</style>