<script>
    import { createEventDispatcher } from "svelte";
    import { timestampToISODate } from "../Common/DateUtils";
    import { InProgressStatuses, StatusButtonCSSClassMap, TestStatusChangeable } from "../Common/TestStatus";
    import { sendMessage } from "../Stores/AlertStore";
    export let testRun;

    const dispatch = createEventDispatcher();
    let disableButtons = false;

    const handleStatus = async function (newStatus) {
        disableButtons = true;
        try {
            let apiResponse = await fetch(`/api/v1/test/${testRun.test_id}/run/${testRun.id}/status/set`, {
                method: "POST",
                headers: {
                    "Content-Type": "application/json",
                },
                body: JSON.stringify({
                    status: newStatus,
                }),
            });
            let apiJson = await apiResponse.json();
            console.log(apiJson);
            if (apiJson.status === "ok") {
                dispatch("statusUpdate", { status: newStatus });
            } else {
                throw apiJson;
            }
        } catch (error) {
            if (error?.status === "error") {
                sendMessage(
                    "error",
                    `API Error updating test run status.\nMessage: ${error.response.arguments[0]}`
                );
            } else {
                sendMessage(
                    "error",
                    "A backend error occurred during test run status update"
                );
            }
        } finally {
            disableButtons = false;
        }
    };
</script>

<div class="dropdown">
    <button
        class="btn {StatusButtonCSSClassMap[
            testRun.status
        ]} text-light"
        type="button"
        title={timestampToISODate(
            testRun.end_time ,
            true
        )}
        data-bs-toggle="dropdown"
    >
        {testRun.status.toUpperCase()}
        {#if InProgressStatuses.find((status) => status == testRun.status)}
            <span
                class="spinner-border spinner-border-sm d-inline-block"
            />
        {/if}
    </button>
    <ul class="dropdown-menu">
        {#each Object.keys(TestStatusChangeable) as status}
            <li>
                <button
                    class="dropdown-item"
                    disabled={disableButtons}
                    on:click={() => {
                        handleStatus(status.toLowerCase());
                    }}>{status}</button
                >
            </li>
        {/each}
    </ul>
</div>

<style>

</style>
