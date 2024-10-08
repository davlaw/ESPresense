<script>
    import SvelteTable from "svelte-table";
    import { configs, events, enroll, cancelEnroll } from "../stores";
    let name = "";
    let id = "";
    let deviceType = "";
    let showModal = false;

    const deviceTypes = ["watch", "wallet", "ipad", "phone", "airpods", "laptop", "node", "keys", "therm", "flora", "tile"];

    function generateKebabCaseId(name, type = "") {
        const words = name.toLowerCase().split(/\s+/);
        const filteredWords = words.filter((word) => word !== type.toLowerCase());
        return filteredWords
            .join("-")
            .replace(/[']/g, "")
            .replace(/[^a-z0-9-]+/g, "-")
            .replace(/(^-|-$)/g, "");
    }

    $: modalTitle = $events?.room ? `Enroll Device - ${$events.room}` : "Enroll Device";
    $: autoGeneratedId = generateKebabCaseId(name);

    function onEnroll() {
        const generatedId = deviceType ? `${deviceType}:${generateKebabCaseId(name, deviceType)}` : generateKebabCaseId(name);
        enroll(id || generatedId, name);
        showModal = false;
    }

    function onClose() {
        showModal = false;
        name = "";
        id = "";
        deviceType = undefined;
        cancelEnroll();
    }

    function formatRemainingTime(seconds) {
        const minutes = Math.floor(seconds / 60);
        const remainingSeconds = seconds % 60;
        return [minutes, remainingSeconds].map((val) => val.toString().padStart(2, "0")).join(":");
    }

    $: if ($events?.state?.enrolling) showModal = true;

    var filterSelections = {};
    var sortBy = "alias";
    var sortOrder = 1;
    var selectedRowIds = [];
    const columns = [
        {
            key: "alias",
            title: "Alias",
            value: (v) => v.alias,
            sortable: true,
            filterOptions: (rows) => {
                const prefixes = new Set();
                rows.forEach((row) => {
                    var prefix = row.alias?.substring(0, row.alias.indexOf(":") + 1);
                    if (prefix?.length > 0) {
                        prefixes.add(prefix);
                    }
                });
                return Array.from(prefixes)
                    .sort()
                    .map((a) => ({ name: a, value: a }));
            },
            filterValue: (v) => v.alias.substring(0, v.alias.indexOf(":") + 1),
            headerClass: "text-left px-6 py-3",
        },
        {
            key: "id",
            title: "ID",
            value: (v) => v.id,
            sortable: true,
            filterOptions: (rows) => {
                const prefixes = new Set();
                rows.forEach((row) => {
                    var prefix = row.id?.substring(0, row.id.indexOf(":") + 1);
                    if (prefix?.length > 0) {
                        prefixes.add(prefix);
                    }
                });
                return Array.from(prefixes)
                    .sort()
                    .map((a) => ({ name: a, value: a }));
            },
            filterValue: (v) => v.id?.substring(0, v.id.indexOf(":") + 1),
            headerClass: "text-left px-6 py-3",
        },
        {
            key: "name",
            title: "Name",
            value: (v) => v.name ?? "",
            sortable: true,
            filterOptions: (rows) => {
                let letrs = {};
                rows.forEach((row) => {
                    let letr = row.name?.charAt(0);
                    if (letr && letrs[letr] === undefined)
                        letrs[letr] = {
                            name: `${letr.toUpperCase()}`,
                            value: letr.toLowerCase(),
                        };
                });
                // fix order
                letrs = Object.entries(letrs)
                    .sort()
                    .reduce((o, [k, v]) => ((o[k] = v), o), {});
                return Object.values(letrs);
            },
            filterValue: (v) => v.name?.charAt(0).toLowerCase(),
            headerClass: "text-left px-6 py-3",
        },
        {
            key: "rssi@1m",
            title: "Rssi@1m",
            value: (v) => v["rssi@1m"],
            renderValue: (v) => (v["rssi@1m"] != null ? `${v["rssi@1m"]} dBm` : ""),
            sortable: true,
            headerClass: "text-left px-6 py-3",
        },
    ];
    function classNameRow(event) {
        event.close ? "bg-yellow-100 row" : "row";
    }
</script>

<button on:click={() => (showModal = true)} class="text-white bg-blue-700 hover:bg-blue-800 focus:ring-4 focus:outline-none focus:ring-blue-300 font-medium rounded-lg text-sm px-5 py-2.5 text-center dark:bg-blue-600 dark:hover:bg-blue-700 dark:focus:ring-blue-800">Enroll</button>
{#if showModal}
    <div class="fixed inset-0 bg-gray-500 bg-opacity-75 transition-opacity">
        <div class="flex items-center justify-center w-full h-full">
            <div class="relative bg-white rounded-lg shadow dark:bg-gray-700 min-w-lg sm:max-w-lg sm:w-full mx-auto">
                <div class="flex justify-between items-start p-4 border-b dark:border-gray-600">
                    <h3 class="text-xl font-semibold text-gray-900 dark:text-white">{modalTitle}</h3>
                    <button type="button" class="text-gray-400 bg-transparent hover:bg-gray-200 hover:text-gray-900 rounded-lg text-sm p-2 ml-auto inline-flex items-center justify-center dark:hover:bg-gray-600 dark:hover:text-white" on:click={onClose}>
                        <span class="sr-only">Close</span>
                    </button>
                </div>
                <div class="p-6 space-y-6">
                    {#if !$events?.state?.enrolling}
                        {#if $events?.state?.enrolledId}
                            <p class="text-sm leading-5 text-gray-500">
                                The device has been enrolled. ESPresense has disconnected; you can delete the pairing on your device. The new device ID is:
                                <span class="font-semibold">{$events.state.enrolledId}</span>.
                            </p>
                        {:else}
                            <div class="mt-3 text-center sm:mt-0 sm:text-left">
                                <select bind:value={deviceType} class="border p-2 w-full mb-2">
                                    <option value="">Select device type</option>
                                    {#each deviceTypes as type}
                                        <option value={type}>{type}</option>
                                    {/each}
                                </select>
                                <input class="border p-2 w-full mb-2" type="text" bind:value={name} placeholder="Enter name" />
                                <input class="border p-2 w-full" type="text" bind:value={id} placeholder={name ? (deviceType ? `${deviceType}:${generateKebabCaseId(name, deviceType)}` : generateKebabCaseId(name)) : "Enter custom ID or leave empty for auto-generated"} />
                                <p class="text-sm text-gray-600 mt-1 mb-2">Leave empty to use auto-generated ID</p>
                            </div>
                        {/if}
                    {:else}
                        <p class="text-sm leading-5 text-gray-500">To begin, navigate to your device's Bluetooth settings (on Apple Watch use the BluetoothLE app) and pair with the ESPresense device.</p>
                        <p class="mt-4 text-sm leading-5 text-gray-500">Start the pairing process now. Time remaining: {formatRemainingTime($events.state.remain)}</p>
                    {/if}
                </div>
                <div class="items-center p-6 space-x-4 space-x-reverse border-t border-gray-200 dark:border-gray-600 sm:px-6 sm:flex sm:flex-row-reverse">
                    {#if !$events?.state?.enrolling && !$events?.state?.enrolledId}
                        <button on:click={onEnroll} disabled={!name} class="text-white bg-blue-700 hover:bg-blue-800 focus:ring-4 focus:outline-none focus:ring-blue-300 font-medium rounded-lg text-sm px-5 py-2.5 text-center dark:bg-blue-600 dark:hover:bg-blue-700 dark:focus:ring-blue-800 disabled:opacity-50 disabled:cursor-not-allowed">Enroll</button>
                    {/if}
                    <button on:click={onClose} class="text-gray-500 bg-white hover:bg-gray-100 focus:ring-4 focus:outline-none focus:ring-blue-300 rounded-lg border border-gray-200 text-sm font-medium px-5 py-2.5 hover:text-gray-900 focus:z-10 dark:bg-gray-700 dark:text-gray-300 dark:border-gray-500 dark:hover:text-white dark:hover:bg-gray-600 dark:focus:ring-gray-600">Close</button>
                </div>
            </div>
        </div>
    </div>
{/if}

<main>
    {#if $configs?.configs != null}
        <SvelteTable {columns} rows={$configs.configs} rowKey="id" bind:filterSelections bind:sortBy bind:sortOrder selectSingle={true} selectOnClick={true} selected={selectedRowIds} classNameTable="min-w-full divide-y divide-gray-200 table-auto" classNameThead="whitespace-nowrap text-left text-xs font-medium text-gray-500 uppercase" classNameTbody="bg-white divide-y divide-gray-200" {classNameRow} classNameRowSelected="bg-blue-100" classNameCell="px-3 py-1 whitespace-no-wrap text-sm leading-5 font-light text-gray-900" classNameInput="px-1 py-1 border rounded-md text-sm leading-5 font-medium text-gray-900 placeholder-gray-500 focus:outline-none focus:shadow-outline-blue focus:border-blue-300 focus:z-10 sm:text-sm sm:leading-5" classNameSelect="px-1 py-1 border rounded-md text-sm leading-5 font-medium text-gray-900 placeholder-gray-500 focus:outline-none focus:shadow-outline-blue focus:border-blue-300 focus:z-10 sm:text-sm sm:leading-5" />
    {:else}
        <h1>Loading configured devices...</h1>
    {/if}
</main>

<style>
    main {
        padding: 1rem;
    }
    h1 {
        text-align: center;
    }
</style>
