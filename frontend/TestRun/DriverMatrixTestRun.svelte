    <script>
    import { onMount, onDestroy, createEventDispatcher } from "svelte";
    import Fa from "svelte-fa";
    import {
        faTimes,
    } from "@fortawesome/free-solid-svg-icons";
    import ActivityTab from "./ActivityTab.svelte";
    import TestRunComments from "./TestRunComments.svelte";
    import GithubIssues from "../Github/GithubIssues.svelte";
    import { sendMessage } from "../Stores/AlertStore";
    import { fetchRun } from "../Common/RunUtils";
    import RunStatusButton from "./RunStatusButton.svelte";
    import RunInvestigationStatusButton from "./RunInvestigationStatusButton.svelte";
    import RunAssigneeSelector from "./RunAssigneeSelector.svelte";
    import DriverMatrixRunInfo from "./DriverMatrixRunInfo.svelte";
    import DriverMatrixTestCollection from "./DriverMatrixTestCollection.svelte";

    export let runId = "";
    export let buildNumber = -1;
    export let testInfo = {};
    const dispatch = createEventDispatcher();
    let testRun = undefined;
    let runRefreshInterval;
    let activityOpen = false;
    let commentsOpen = false;
    let issuesOpen = false;
    let failedToLoad = false;

    const fetchTestRunData = async function () {
        try {
            let run = await fetchRun(testInfo.test.plugin_name, runId);
            testRun = run;
            if (!testRun) {
                failedToLoad = true;
                return;
            }
            if (buildNumber == -1) {
                buildNumber = parseInt(
                    testRun.build_job_url.split("/").reverse()[1]
                );
            }
        } catch (error) {
            if (error?.status === "error") {
                sendMessage(
                    "error",
                    `API Error when fetching test run data.\nMessage: ${error.response.arguments[0]}`
                );
            } else {
                sendMessage(
                    "error",
                    "A backend error occurred during test run data fetch"
                );
            }
        }
    };

    onMount(() => {
        fetchTestRunData();

        runRefreshInterval = setInterval(() => {
            fetchTestRunData();
        }, 1000 * 30);
    });

    onDestroy(() => {
        if (runRefreshInterval) clearInterval(runRefreshInterval);
    });
</script>

<div class="border rounded shadow-sm testrun-card mb-4 top-bar">
    <div class="d-flex px-2 py-2 mb-1 border-bottom bg-white ">
        <div class="p-1">
            {#if testRun}
                <a class="link-dark" href="/test/{testRun.test_id}/runs?additionalRuns[]={testRun.id}">
                    {testRun.build_id}#{buildNumber}
                </a>
            {/if}
        </div>
        <div class="ms-auto text-end">
            <button class="btn btn-sm btn-outline-dark" title="Close" on:click={() => {
                dispatch("closeRun", { id: runId });
            }}
                ><Fa icon={faTimes} /></button
            >
        </div>
    </div>
    {#if testRun}
        <div class="p-2">
            <div class="row p-2">
                <div class="col-6">
                    <div class="d-flex align-items-center">
                        <RunStatusButton {testRun} on:statusUpdate={(e) => {
                            testRun.status = e.detail.status;
                        }} />
                        <RunInvestigationStatusButton {testRun} on:statusUpdate={(e) => {
                            testRun.investigation_status = e.detail.status;
                        }} />
                    </div>
                </div>
                <div class="col-6">
                    <RunAssigneeSelector {testRun} on:assigneeUpdate={(e) => {
                        testRun.assignee = e.detail.assignee;
                    }} />
                </div>
            </div>
            <nav>
                <div class="nav nav-tabs" id="nav-tab-{runId}" role="tablist">
                    <button
                        class="nav-link active"
                        id="nav-details-tab-{runId}"
                        data-bs-toggle="tab"
                        data-bs-target="#nav-details-{runId}"
                        type="button"
                        role="tab"
                        ><i class="fas fa-info-circle" /> Details</button
                    >
                    <button
                        class="nav-link"
                        id="nav-tests-tab-{runId}"
                        data-bs-toggle="tab"
                        data-bs-target="#nav-tests-{runId}"
                        type="button"
                        role="tab"
                        ><i class="fas fa-boxes" /> Tests</button
                    >
                    <button
                        class="nav-link"
                        id="nav-discuss-tab-{runId}"
                        data-bs-toggle="tab"
                        data-bs-target="#nav-discuss-{runId}"
                        type="button"
                        on:click={() => (commentsOpen = true)}
                        role="tab"
                        ><i class="fas fa-comments" /> Discussion</button
                    >
                    <button
                        class="nav-link"
                        id="nav-issues-tab-{runId}"
                        data-bs-toggle="tab"
                        data-bs-target="#nav-issues-{runId}"
                        type="button"
                        role="tab"
                        on:click={() => (issuesOpen = true)}
                        ><i class="fas fa-code-branch" /> Issues</button
                    >
                    <button
                        class="nav-link"
                        id="nav-activity-tab-{runId}"
                        data-bs-toggle="tab"
                        data-bs-target="#nav-activity-{runId}"
                        type="button"
                        on:click={() => (activityOpen = true)}
                        role="tab"
                        ><i class="fas fa-exclamation-triangle" /> Activity</button
                    >
                </div>
            </nav>
            <div
                class="tab-content border-start border-end border-bottom bg-white"
                id="nav-tabContent-{runId}"
            >
                <div
                    class="tab-pane fade show active"
                    id="nav-details-{runId}"
                    role="tabpanel"
                >
                    <DriverMatrixRunInfo testRun={testRun} {testInfo}/>
                </div>
                <div
                    class="tab-pane fade"
                    id="nav-tests-{runId}"
                    role="tabpanel"
                >
                    <DriverMatrixTestCollection testCollection={testRun.test_collection} testId={testRun.id}/>
                </div>
                <div
                    class="tab-pane fade"
                    id="nav-discuss-{runId}"
                    role="tabpanel"
                >
                    {#if commentsOpen}
                        <TestRunComments {testRun} {testInfo}/>
                    {/if}
                </div>
                <div class="tab-pane fade" id="nav-issues-{runId}" role="tabpanel">
                    <div class="py-2 bg-white">
                        {#if issuesOpen}
                            <GithubIssues id={runId} testId={testInfo.test.id} />
                        {/if}
                    </div>
                </div>
                <div
                    class="tab-pane fade"
                    id="nav-activity-{runId}"
                    role="tabpanel"
                >
                    {#if activityOpen}
                        <ActivityTab id={runId} />
                    {/if}
                </div>
            </div>
        </div>
    {:else if failedToLoad}
        <div
            class="text-center p-2 m-1 d-flex align-items-center justify-content-center"
        >
            <span class="fs-4"
                >Run not found.</span
            >
        </div>
    {:else}
        <div
            class="text-center p-2 m-1 d-flex align-items-center justify-content-center"
        >
            <span class="spinner-border me-4" /><span class="fs-4"
                >Loading...</span
            >
        </div>
    {/if}
</div>

<style>
    .testrun-card {
        background-color: #ededed;
    }

    .top-bar {
        overflow: hidden;
    }
</style>
