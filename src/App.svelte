<script>
    import Router from "svelte-spa-router";
    import Main from "./Main.svelte";

    const routes = {
        "/:network?/:contract?": Main,
    };
</script>

<Router {routes} />
