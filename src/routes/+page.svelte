<script>
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';

    // --- Time Helpers ---

    function getCurrentDateTimeLocalString() {
        const now = new Date();
        const offset = now.getTimezoneOffset() * 60000;
        const localISOTime = (new Date(now.getTime() - offset)).toISOString().slice(0, 16);
        return localISOTime;
    }

     function formatDateTimeForDisplay(dateTimeString) {
        if (!dateTimeString) return '';
        try {
            const date = new Date(dateTimeString);
            if (isNaN(date.getTime())) throw new Error("Invalid date");
            return new Intl.DateTimeFormat(navigator.language, {
                year: 'numeric', month: 'short', day: 'numeric',
                hour: 'numeric', minute: '2-digit'
            }).format(date);
        } catch (e) {
            console.error("Error formatting date:", e);
            return "Invalid Date";
        }
    }

    /**
     * Calculates the duration since a start date/time up to a given 'now' date/time.
     * @param {string} startDateTimeString - The starting date/time string.
     * @param {Date} nowDate - The current date/time to calculate duration up to.
     * @returns {{days: number, hours: number, minutes: number, totalMinutes: number } | null}
     */
    function calculateDurationSince(startDateTimeString, nowDate) {
        if (!startDateTimeString || !nowDate) return null;
        try {
            const startDate = new Date(startDateTimeString);
            if (isNaN(startDate.getTime())) throw new Error("Invalid start date");

            let diffMs = nowDate.getTime() - startDate.getTime();
            diffMs = Math.max(0, diffMs); // Ensure non-negative

            const msInMinute = 60 * 1000;
            const msInHour = 60 * msInMinute;
            const msInDay = 24 * msInHour;

            const totalMinutes = Math.floor(diffMs / msInMinute);
            const days = Math.floor(diffMs / msInDay);
            const remainingMsAfterDays = diffMs % msInDay;
            const hours = Math.floor(remainingMsAfterDays / msInHour);
            const remainingMsAfterHours = remainingMsAfterDays % msInHour;
            const minutes = Math.floor(remainingMsAfterHours / msInMinute);

            return { days, hours, minutes, totalMinutes };

        } catch (error) {
            console.error("Error calculating duration:", error);
            return null;
        }
    }

     function formatDuration(duration) {
        if (!duration) return "Calculating...";
        let parts = [];
        if (duration.days > 0) parts.push(`${duration.days}d`);
        if (duration.hours > 0) parts.push(`${duration.hours}h`);
        // Always show minutes unless only days/hours are present and minutes are 0
        if (duration.minutes >= 0 && (duration.days > 0 || duration.hours > 0 || (duration.days === 0 && duration.hours === 0))) {
             parts.push(`${duration.minutes}m`);
        }
        return parts.join(' ') || '0m';
     }

	// --- State ---
	let items = [];
	let newItemName = '';
	let newItemDateTime = getCurrentDateTimeLocalString();
	let isLoading = true;
	const LOCAL_STORAGE_KEY = 'daysSinceItems_svelte98_dt_v1';

    // --- State for Timer ---
    let currentTime = new Date(); // Reactive variable holding current time
    let intervalId = null;      // To store the interval ID for cleanup

	// --- Lifecycle & Persistence ---
	onMount(() => {
        if (browser) {
            // Load initial data
            const savedItems = localStorage.getItem(LOCAL_STORAGE_KEY);
            if (savedItems) {
                try {
                    const parsedItems = JSON.parse(savedItems);
                    items = Array.isArray(parsedItems) ? parsedItems : [];
                } catch (e) { console.error("Failed localStorage parse:", e); items = []; }
            }
            isLoading = false;

            // --- Start the auto-refresh timer ---
            // Update currentTime every minute (60 * 1000 milliseconds)
            intervalId = setInterval(() => {
                currentTime = new Date(); // This assignment triggers reactivity
            }, 60 * 1000);

        } else {
             isLoading = false;
        }

        // --- Cleanup function ---
        // This runs when the component is destroyed
        return () => {
            if (browser && intervalId) {
                clearInterval(intervalId); // Stop the timer
                // console.log('Timer cleared'); // For debugging
            }
        };
	}); // <-- End of onMount

	// --- Reactive statement to save items ---
	$: {
		if (browser && !isLoading) {
            const validItems = items.filter(item => item && item.dateTimeString);
			localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(validItems));
		}
	}

    // --- Reactive calculation of durations ---
    // This $: block re-runs whenever 'items' OR 'currentTime' changes
    $: itemDurations = items.map(item => ({
        id: item.id,
        // Pass the reactive currentTime to the calculation
        duration: calculateDurationSince(item.dateTimeString, currentTime)
    }));

    // --- Helper to get formatted duration for an item ID ---
    // This function doesn't need to be reactive itself, it just reads
    // the reactive 'itemDurations' array.
    function getFormattedDuration(itemId) {
        const itemDurationData = itemDurations.find(d => d.id === itemId);
        return itemDurationData ? formatDuration(itemDurationData.duration) : 'Error';
    }


	// --- Event Handlers (Remain the same) ---
	function handleAddItem() {
        const name = newItemName.trim();
		const dateTimeString = newItemDateTime;
		if (name && dateTimeString) {
			const newItem = { id: Date.now(), name, dateTimeString };
			items = [...items, newItem];
			newItemName = '';
			newItemDateTime = getCurrentDateTimeLocalString();
		} else { if (browser) alert("Please enter name and select date/time."); }
    }
    function handleDeleteItem(idToDelete) { items = items.filter(item => item.id !== idToDelete); }
    function handleRestartItem(idToRestart) {
        const nowDateTimeString = getCurrentDateTimeLocalString();
        items = items.map(item => item.id === idToRestart ? { ...item, dateTimeString: nowDateTimeString } : item );
    }

</script>

<svelte:head>
    <title>Days Since 98</title>
</svelte:head>

<!-- Main Window Structure -->
<div class="main-window-container">
    <div class="window">
        <div class="title-bar">
             <div class="title-bar-text">Days Since Tracker 98</div>
             <div class="title-bar-controls">
                 <button aria-label="Minimize"></button> <button aria-label="Maximize"></button> <button aria-label="Close"></button>
             </div>
        </div>
        <div class="window-body">
            {#if isLoading}
                <p class="empty-list-message">Loading...</p>
            {:else}
                <!-- Add Item Form -->
                <form class="add-form" on:submit|preventDefault={handleAddItem}>
                    <fieldset>
                        <legend>Add New Item</legend>
                        <div class="inline-form-row">
                            <input id="itemName" type="text" placeholder="Event" bind:value={newItemName} required aria-label="Event Name"/>
                            <input id="itemDateTime" type="datetime-local" bind:value={newItemDateTime} required max={getCurrentDateTimeLocalString()} aria-label="Start Date and Time"/>
                            <button type="submit">Add</button>
                        </div>
                    </fieldset>
                </form>

                <!-- Item List - Uses getFormattedDuration -->
                 {#if items.length > 0}
                    <ul class="tree-view">
                        <!-- Use item.id as the key -->
                        {#each items as item (item.id)}
                            <li class="compact-item">
                                <details class="item-summary-details" open={false}>
                                    <summary>
                                        <!-- Call helper function to display reactive duration -->
                                        <span class="duration">{getFormattedDuration(item.id)}</span>
                                        <span class="event-separator"> since </span>
                                        <span class="event-name">{item.name}</span>
                                    </summary>
                                    <div class="item-start-date">
                                        <p>Started: {formatDateTimeForDisplay(item.dateTimeString)}</p>
                                    </div>
                                </details>
                                <div class="item-controls">
                                     <button type="button" on:click={() => handleRestartItem(item.id)} aria-label={`Restart ${item.name}`}>Restart</button>
                                     <button type="button" on:click={() => handleDeleteItem(item.id)} aria-label={`Delete ${item.name}`}>Delete</button>
                                </div>
                            </li>
                        {/each}
                    </ul>
                 {:else}
                     <p class="empty-list-message">No items yet. Add something above!</p>
                 {/if}

            {/if}
        </div> <!-- /window-body -->
    </div> <!-- /window -->
</div> <!-- /main-window-container -->