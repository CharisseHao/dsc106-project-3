<script>
    let tempData = [];

    onMount(async () => {
        const res = await fetch('yelp_academic_dataset_review.json'); 
        const csv = await res.text();
        tempData = d3.csvParse(csv, d3.autoType)
        console.log(tempData);
    });

</script>

<main>

</main>

<style>

</style>
