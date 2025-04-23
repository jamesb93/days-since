<script>
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';

    // --- Time Helpers ---

    /**
     * Gets the current date and time formatted for datetime-local input.
     * Format: YYYY-MM-DDTHH:mm (seconds not supported by input)
     */
    function getCurrentDateTimeLocalString() {
        const now = new Date();
        // Adjust for local timezone offset
        const offset = now.getTimezoneOffset() * 60000; // Offset in milliseconds
        const localISOTime = (new Date(now.getTime() - offset)).toISOString().slice(0, 16);
        return localISOTime;
    }

     /**
      * Formats an ISO-like date-time string for display.
      * @param {string} dateTimeString (e.g., from datetime-local input or ISO)
      * @returns {string} Formatted string (e.g., "YYYY-MM-DD hh:mm AM/PM") or "Invalid Date"
      */
     function formatDateTimeForDisplay(dateTimeString) {
        if (!dateTimeString) return '';
        try {
            const date = new Date(dateTimeString);
            if (isNaN(date.getTime())) throw new Error("Invalid date");
            // Use Intl.DateTimeFormat for locale-aware formatting
            return new Intl.DateTimeFormat(navigator.language, { // Use browser's locale
                year: 'numeric',
                month: 'short',
                day: 'numeric',
                hour: 'numeric',
                minute: '2-digit',
                // hour12: true // Optional: force 12-hour clock
            }).format(date);
        } catch (e) {
            console.error("Error formatting date:", e);
            return "Invalid Date";
        }
    }


    /**
     * Calculates the duration (days, hours, minutes) since a given date-time string.
     * @param {string} dateTimeString (ISO format or compatible with new Date())
     * @returns {{days: number, hours: number, minutes: number, totalMinutes: number }} Duration object or null on error
     */
    function calculateDurationSince(dateTimeString) {
        if (!dateTimeString) return null;
        try {
            const startDate = new Date(dateTimeString);
            if (isNaN(startDate.getTime())) throw new Error("Invalid start date");

            const now = new Date();
            let diffMs = now.getTime() - startDate.getTime();

            // Ensure difference is not negative (if start date is in future)
            diffMs = Math.max(0, diffMs);

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
            return null; // Indicate error
        }
    }

     /**
      * Formats the duration object into a readable string.
      * @param {{days: number, hours: number, minutes: number}} duration
      * @returns {string} Formatted string e.g., "5d 3h 15m"
      */
     function formatDuration(duration) {
        if (!duration) return "Calculating...";
        // Simple format: 5d 3h 15m
        // return `${duration.days}d ${duration.hours}h ${duration.minutes}m`;

        // More descriptive format:
        let parts = [];
        if (duration.days > 0) parts.push(`${duration.days} day${duration.days > 1 ? 's' : ''}`);
        if (duration.hours > 0) parts.push(`${duration.hours} hr${duration.hours > 1 ? 's' : ''}`);
        // Always show minutes unless days/hours are present and minutes are 0
        if (duration.minutes > 0 || parts.length === 0) {
             parts.push(`${duration.minutes} min${duration.minutes > 1 ? 's' : ''}`);
        }

        return parts.join(', ') || '0 minutes'; // Handle case where duration is exactly 0
     }


	// --- State ---
	let items = [];
	let newItemName = '';
	// Use the new helper for the default value
	let newItemDateTime = getCurrentDateTimeLocalString();
	let isLoading = true;
    // Use a new key because the data format changed
	const LOCAL_STORAGE_KEY = 'daysSinceItems_svelte98_dt_v1';
    // Reactive timer to update durations every minute
    let currentTime = new Date();
    let intervalId = null;


	// --- Lifecycle & Persistence ---
	onMount(() => {
        if (browser) {
            const savedItems = localStorage.getItem(LOCAL_STORAGE_KEY);
            if (savedItems) {
                try {
                    const parsedItems = JSON.parse(savedItems);
                    // Add validation for the new dateTimeString property if needed
                    items = Array.isArray(parsedItems) ? parsedItems : [];
                } catch (e) { console.error("Failed localStorage parse:", e); items = []; }
            }
             isLoading = false;

             // Start timer to update displayed durations
             intervalId = setInterval(() => {
                currentTime = new Date(); // Trigger reactivity by updating time
             }, 60 * 1000); // Update every minute
        } else { isLoading = false; }

        // Cleanup timer on component destroy
        return () => {
            if (intervalId) clearInterval(intervalId);
        }
	});

	$: { // Save to localStorage reactively
		if (browser && !isLoading) {
            // Filter out any potential invalid items before saving
            const validItems = items.filter(item => item && item.dateTimeString);
			localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(validItems));
		}
	}

	// --- Event Handlers ---
	function handleAddItem() {
        const name = newItemName.trim();
		const dateTimeString = newItemDateTime; // Value from datetime-local input
		if (name && dateTimeString) {
            // Basic validation of the datetime string format if needed,
            // but browsers usually enforce it for the input type.
			const newItem = {
                id: Date.now(),
                name,
                // Store the value directly from datetime-local input
                // It's compatible with new Date() but isn't strictly ISO 8601 with Z offset
                dateTimeString: dateTimeString
            };
			items = [...items, newItem];
			newItemName = '';
			newItemDateTime = getCurrentDateTimeLocalString(); // Reset to current time
		} else { if (browser) alert("Please enter name and select date/time."); }
    }

    function handleDeleteItem(idToDelete) {
        items = items.filter(item => item.id !== idToDelete);
    }

    function handleRestartItem(idToRestart) {
        // Get the current time formatted for storage/display
        const nowDateTimeString = getCurrentDateTimeLocalString();
        items = items.map(item =>
            item.id === idToRestart
                ? { ...item, dateTimeString: nowDateTimeString } // Update the dateTimeString
                : item
        );
    }

    // Reactive calculation of durations for display
    // This recalculates whenever 'items' or 'currentTime' changes
    $: itemDurations = items.map(item => ({
        id: item.id,
        duration: calculateDurationSince(item.dateTimeString)
    }));

    // Helper to get duration string for a specific item ID
    function getFormattedDuration(itemId) {
        // Need access to currentTime to ensure reactivity
        const _ = currentTime;
        const itemDurationData = itemDurations.find(d => d.id === itemId);
        return itemDurationData ? formatDuration(itemDurationData.duration) : 'Error';
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
                 <button aria-label="Minimize"></button>
                 <button aria-label="Maximize"></button>
                 <button aria-label="Close"></button>
             </div>
        </div>
        <div class="window-body">
            {#if isLoading}
                <p class="empty-list-message">Loading...</p>
            {:else}
                <!-- Add Item Form - Updated for Inline Layout & DateTime -->
                <form class="add-form" on:submit|preventDefault={handleAddItem}>
                    <fieldset>
                        <legend>Add New Item</legend>
                        <div class="inline-form-row">
                            <input
                                id="itemName"
                                type="text"
                                placeholder="Event"
                                bind:value={newItemName}
                                required
                                aria-label="Event Name"
                            />
                            <!-- Use datetime-local input -->
                            <input
                                id="itemDateTime"
                                type="datetime-local"
                                bind:value={newItemDateTime}
                                required
                                max={getCurrentDateTimeLocalString()}
                                aria-label="Start Date and Time"
                            />
                            <button type="submit">Add</button>
                        </div>
                    </fieldset>
                </form>

                <!-- Item List - Updated Display -->
                 {#if items.length > 0}
                    <ul class="tree-view">
                        {#each items as item (item.id)}
                            <li class="compact-item">
                                <details class="item-summary-details" open={false}>
                                    <summary>
                                        <!-- Display detailed duration -->
                                        <span class="duration">{getFormattedDuration(item.id)}</span>
                                        <!-- Add "since" text -->
                                        <span class="event-separator"> since </span>
                                        <span class="event-name">{item.name}</span>
                                    </summary>
                                    <div class="item-start-date">
                                        <!-- Display formatted start date/time -->
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