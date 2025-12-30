<script lang="ts">
    import { emojiCategories } from "./emoji-list";

    const { onselect = (str: string) => {} } = $props();

    let recentEmojis = $state(["😀", "👍", "❤️", "😂", "🎉", "👋", "🔥", "💯"]);
    let activeCategory = $state("Smileys & People");

    const allEmojis = Object.values(emojiCategories).flat();

    // Optimize by precomputing search terms at initialization
    const categoryIcons = new Map(
        Object.entries({
            "Smileys & People": "😀",
            "Animals & Nature": "🐶",
            "Food & Drink": "🍎",
            Activity: "⚽",
            "Travel & Places": "🚗",
            Objects: "💻",
            Symbols: "❤️",
            Flags: "🏁",
        }),
    );

    function selectEmoji(emoji: string) {
        recentEmojis = [
            emoji,
            ...recentEmojis.filter((e) => e !== emoji),
        ].slice(0, 8);
        onselect(emoji);
    }
</script>

<emoji>
    <div class="header">
        <h3>Pick an emoji</h3>
    </div>

    <!-- Category Tabs -->
    <div class="categories">
        {#each emojiCategories.keys() as category}
            <button
                onclick={() => (activeCategory = category)}
                class="category-btn {activeCategory === category
                    ? 'active'
                    : ''}"
                title={category}
            >
                {categoryIcons.get(category)}
            </button>
        {/each}
    </div>

    <!-- Emoji Grid -->
    <div class="grid-container">
        {#if recentEmojis.length > 0}
            <div class="section">
                <p>Recent</p>
                <div class="grid">
                    {#each recentEmojis as emoji}
                        <button
                            onclick={() => selectEmoji(emoji)}
                            class="emoji-btn"
                        >
                            {emoji}
                        </button>
                    {/each}
                </div>
            </div>
        {/if}

        <div class="section">
            <p>{activeCategory}</p>
            <div class="grid">
                {#each emojiCategories.get(activeCategory) as emoji}
                    <button
                        onclick={() => selectEmoji(emoji)}
                        class="emoji-btn"
                    >
                        {emoji}
                    </button>
                {/each}
            </div>
        </div>
    </div>
</emoji>

<style>
    emoji {
        border: 1px solid #ccc;
        border-radius: 8px;
        background-color: #f8f8f8;
        width: 100%;
        flex-grow: 0;
    }

    .header {
        display: flex;
        align-items: center;
        margin-bottom: 2px;
        border-bottom: 1px solid #ccc;
        flex-direction: column;
    }

    h3 {
        margin: 0.5em;
        font-size: 1rem;
        line-height: 1.5rem;
        font-weight: 600;
        color: #111827;
    }

    button {
        border: none;
        background: none;
    }

    .categories {
        display: flex;
        gap: 0.25rem;
        overflow-x: auto;
        margin-bottom: 0em;
        padding-bottom: 0.2em;
        border-bottom: 1px solid #ccc;
    }

    .category-btn {
        display: flex;
        align-items: center;
        justify-content: center;
        min-width: 2.5rem;
        height: 2.5rem;
        font-size: 1.125rem;
        line-height: 1.75rem;
        border-radius: 0.375rem;
        color: #374151;
    }

    .category-btn:hover {
        background-color: #e5e7eb;
    }

    .category-btn.active {
        background-color: #dbeafe;
        color: #1d4ed8;
    }

    .grid-container {
        height: 14rem;
        overflow-y: auto;
        overflow-x: hidden;
    }

    .section {
        margin-bottom: 0.75em;
    }

    .section p {
        font-size: 0.75rem;
        line-height: 1rem;
        font-weight: 500;
        color: #6b7280;
        text-transform: uppercase;
        margin-bottom: 0.25rem;
    }

    .grid {
        display: flex;
        flex-wrap: wrap;
        flex-grow: 0;
        gap: 0.25rem;
        max-width: 448px;
    }

    .emoji-btn {
        width: 2rem;
        height: 2rem;
        font-size: 1.125rem;
        line-height: 1.75rem;
        border-radius: 0.25rem;
    }

    .emoji-btn:hover {
        background-color: #e5e7eb;
        transform: scale(1.5);
    }

    .emoji-btn:active {
        transform: scale(0.95);
    }
</style>
