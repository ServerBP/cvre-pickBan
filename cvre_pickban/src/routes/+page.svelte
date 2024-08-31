<script>
  // @ts-nocheck
  
    import { onMount } from 'svelte';
  
    let mapKeys = '';
    let mapPool = [];
    let columns = {
      'Team 1 Pick': [],
      'Team 2 Pick': [],
      'Team 1 Ban': [],
      'Team 2 Ban': []
    };
    let team1Name = 'Team 1';
    let team2Name = 'Team 2';
    let isMatchFlowComplete = false;
    let draggedMap = null;
    let expandedMap = null;
    let winnerTeam = null;
    let isDataLoaded = false; // New flag to ensure data is loaded before saving
  
    function saveToLocalStorage() {
      if (!isDataLoaded) return; // Avoid saving if data isn't loaded yet
  
      localStorage.setItem('mapPool', JSON.stringify(mapPool));
      localStorage.setItem('columns', JSON.stringify(columns));
      localStorage.setItem('team1Name', team1Name);
      localStorage.setItem('team2Name', team2Name);
      localStorage.setItem('winnerTeam', winnerTeam);
    }
  
    function loadFromLocalStorage() {
      const storedMapPool = localStorage.getItem('mapPool');
      const storedColumns = localStorage.getItem('columns');
      const storedTeam1Name = localStorage.getItem('team1Name');
      const storedTeam2Name = localStorage.getItem('team2Name');
      const storedWinnerTeam = localStorage.getItem('winnerTeam');
  
      if (storedMapPool) mapPool = JSON.parse(storedMapPool);
      if (storedColumns) columns = JSON.parse(storedColumns);
      if (storedTeam1Name) team1Name = storedTeam1Name;
      if (storedTeam2Name) team2Name = storedTeam2Name;
      if (storedWinnerTeam) winnerTeam = storedWinnerTeam;
  
      // Ensure columns object has all required properties
      columns = {
        [`${team1Name} Pick`]: columns[`${team1Name} Pick`] || [],
        [`${team2Name} Pick`]: columns[`${team2Name} Pick`] || [],
        [`${team1Name} Ban`]: columns[`${team1Name} Ban`] || [],
        [`${team2Name} Ban`]: columns[`${team2Name} Ban`] || []
      };
  
      isDataLoaded = true; // Data is now loaded, allow saving
    }
  
    onMount(() => {
      loadFromLocalStorage();
      checkMatchFlowComplete();
    });
  
    // Reactive block to save changes to localStorage
    $: {
      if (isDataLoaded) {
        updateWinnerTeam();
        saveToLocalStorage();
      }
    }
  
    function updateWinnerTeam() {
      const team1Wins = columns[`${team1Name} Pick`].filter(m => m.winner === team1Name).length;
      const team2Wins = columns[`${team2Name} Pick`].filter(m => m.winner === team2Name).length;
      if (team1Wins > team2Wins) {
        winnerTeam = team1Name;
      } else if (team2Wins > team1Wins) {
        winnerTeam = team2Name;
      } else {
        winnerTeam = null;
      }
    }
  
    async function addMaps() {
      if (!mapKeys) return;
      const keys = mapKeys.split(',').map(key => key.trim()).filter(key => key !== '');
      
      for (const key of keys) {
        try {
          const response = await fetch(`https://api.beatsaver.com/maps/id/${key}`);
          const mapData = await response.json();
          mapPool = [...mapPool, {
            key: key,
            name: mapData.name,
            image: mapData.versions[0].coverURL,
            difficulties: mapData.versions[0].diffs.map(d => d.difficulty),
            selectedDifficulty: mapData.versions[0].diffs[0].difficulty,
            result: '',
            winner: null
          }];
        } catch (error) {
          console.error(`Failed to add map with key ${key}:`, error);
        }
      }
      
      mapKeys = '';
      saveToLocalStorage();
    }
  
    function onDragStart(event, map) {
      draggedMap = map;
      event.dataTransfer.setData('text/plain', JSON.stringify(map));
    }
  
    function onDragOver(event) {
      event.preventDefault();
      event.target.closest('.column')?.classList.add('drag-over');
    }
  
    function onDragLeave(event) {
      event.target.closest('.column')?.classList.remove('drag-over');
    }
  
    function onDrop(event, columnName) {
      event.preventDefault();
      const column = event.target.closest('.column');
      column?.classList.remove('drag-over');
      
      if (!draggedMap) return;
      
      Object.keys(columns).forEach(col => {
        columns[col] = columns[col].filter(m => m.key !== draggedMap.key);
      });
      
      columns[columnName] = [...columns[columnName], draggedMap];
      columns = columns; // Trigger reactivity
  
      draggedMap = null;
      checkMatchFlowComplete();
      saveToLocalStorage();
    }
  
    function checkMatchFlowComplete() {
      const picksCount = columns[`${team1Name} Pick`].length + columns[`${team2Name} Pick`].length;
      const bansCount = columns[`${team1Name} Ban`].length + columns[`${team2Name} Ban`].length;
      isMatchFlowComplete = picksCount === 5 && bansCount === 2;
    }
  
    function resetMatch() {
      columns = {
        [`${team1Name} Pick`]: [],
        [`${team2Name} Pick`]: [],
        [`${team1Name} Ban`]: [],
        [`${team2Name} Ban`]: []
      };
      team1Name = 'Team 1';
      team2Name = 'Team 2';
      winnerTeam = null;
      saveToLocalStorage();
      checkMatchFlowComplete();
    }
  
    // Other functions like `createMatchFlow`, `copyMatchData`, `removeMap`, `selectMapWinner`, `copyMapLink`, etc. remain unchanged.


// ---- DEPRECATED FUNCTION - DOESN'T RETURN MAPS IN ORDER ----

//   function createMatchFlow() {
//     const matchFlow = `# ${team1Name} vs ${team2Name}: Match Flow
// ## Maps played:
// ${columns[`${team1Name} Pick`].map(m => `* ${m.name}`).join('\n')}
// ${columns[`${team2Name} Pick`].map(m => `* ${m.name}`).join('\n')}
// * ${columns[`${team2Name} Pick`][2]?.name || 'Tiebreaker'} (Tiebreaker)

// ## Bans:
// * ${columns[`${team1Name} Ban`][0]?.name || 'N/A'} - ${team1Name}
// * ${columns[`${team2Name} Ban`][0]?.name || 'N/A'} - ${team2Name}`;

//     navigator.clipboard.writeText(matchFlow);
//     alert('Match Flow copied to clipboard!');
//   }

function createMatchFlow() {
    const matchFlow = `# ${team1Name} vs ${team2Name}: Match Flow
## Maps played:
1. ${columns[`${team1Name} Pick`][0].name}
2. ${columns[`${team2Name} Pick`][0].name}
3. ${columns[`${team2Name} Pick`][1].name}
4. ${columns[`${team1Name} Pick`][1].name}
5.* ${columns[`${team2Name} Pick`][2]?.name || 'Tiebreaker' || columns[`${team1Name} Pick`][2]?.name} (Tiebreaker)

## Bans:
* ${columns[`${team1Name} Ban`][0]?.name || 'N/A'} - ${team1Name}
* ${columns[`${team2Name} Ban`][0]?.name || 'N/A'} - ${team2Name}`;

    navigator.clipboard.writeText(matchFlow);
    alert('Match Flow copied to clipboard!');
  }

  function removeMap(map) {
    mapPool = mapPool.filter(m => m.key !== map.key);
    Object.keys(columns).forEach(col => {
      columns[col] = columns[col].filter(m => m.key !== map.key);
    });
    columns = columns; // Trigger reactivity
    checkMatchFlowComplete();
    saveToLocalStorage();
  }

  function saveMatchResult(map, result) {
    map.result = result;
    saveToLocalStorage();
  }

  function copyMapLink(mapKey) {
    const link = `https://beatsaver.com/maps/${mapKey}`;
    navigator.clipboard.writeText(link);
    alert('Map link copied to clipboard!');
  }

  function selectMapWinner(map, winner) {
    map.winner = winner;
    saveToLocalStorage();
  }

  function getUpcomingMap() {
    const allMaps = [...columns[`${team1Name} Pick`], ...columns[`${team2Name} Pick`]];
    return allMaps.find(map => !map.winner);
  }

  function copyMatchData() {
    const matchData = `# ${team1Name} vs ${team2Name}: Match Data

## Map Pool:
${mapPool.map(m => `* ${m.name} (${m.key})`).join('\n')}

## Picks and Bans:
${team1Name} Picks: ${columns[`${team1Name} Pick`].map(m => m.name).join(', ')}
${team2Name} Picks: ${columns[`${team2Name} Pick`].map(m => m.name).join(', ')}
${team1Name} Bans: ${columns[`${team1Name} Ban`].map(m => m.name).join(', ')}
${team2Name} Bans: ${columns[`${team2Name} Ban`].map(m => m.name).join(', ')}

## Match Flow:
${[...columns[`${team1Name} Pick`], ...columns[`${team2Name} Pick`]].map((m, i) => 
  `${i + 1}. ${m.name} - ${m.result ? `Result: ${m.result}` : 'No result'} - ${m.winner ? `Winner: ${m.winner}` : 'No winner selected'}`
).join('\n')}

## Tiebreaker:
${columns[`${team2Name} Pick`][2]?.name || 'No tiebreaker set'}

## Winner:
${winnerTeam || 'No winner yet'}`;

navigator.clipboard.writeText(matchData);
alert('Match Data copied to clipboard!');
}
  
  </script>
  

<main>
  <h1>Beat Saber Tournament Manager</h1>
  
  <div class="team-names">
    <input bind:value={team1Name} placeholder="Team 1 Name" />
    <input bind:value={team2Name} placeholder="Team 2 Name" />
  </div>

  {#if winnerTeam}
    <div class="winner-announcement">
      <h2>🏆 Winner: {winnerTeam} 🏆</h2>
    </div>
  {/if}

  <div class="map-input">
    <input bind:value={mapKeys} placeholder="Enter map key" />
    <button on:click={addMaps}>
      <span class="material-icons">add</span>
      Add Map
    </button>
  </div>

  <div class="columns">
    {#each Object.entries(columns) as [columnName, maps]}
      <!-- svelte-ignore a11y-no-static-element-interactions -->
      <div 
        class="column"
        class:ban={columnName.includes('Ban')}
        class:pick={columnName.includes('Pick')}
        on:dragover={onDragOver}
        on:dragleave={onDragLeave}
        on:drop={(e) => onDrop(e, columnName)}
      >
        <h2>{columnName}</h2>
        {#each maps as map}
          <!-- svelte-ignore a11y-click-events-have-key-events -->
          <!-- svelte-ignore a11y-no-static-element-interactions -->
          <div class="map-card" class:expanded={expandedMap === map.key}>
            <!-- svelte-ignore a11y-click-events-have-key-events -->
            <!-- svelte-ignore a11y-no-static-element-interactions -->
            <div class="map-header" on:click={() => expandedMap = expandedMap === map.key ? null : map.key}>
              <img src={map.image} alt={map.name} />
              <div class="map-info">
                <div class="map-name">{map.name}</div>
                <select bind:value={map.selectedDifficulty} on:click={(e) => e.stopPropagation()}>
                  {#each map.difficulties as difficulty}
                    <option value={difficulty}>{difficulty}</option>
                  {/each}
                </select>
              </div>
            </div>
            {#if expandedMap === map.key}
              <div class="map-details">
                <input 
                  type="text" 
                  placeholder="Enter match result" 
                  bind:value={map.result}
                  on:input={(e) => saveMatchResult(map, e.target.value)}
                />
                <div class="map-actions">
                  <button on:click={() => copyMapLink(map.key)}>
                    <span class="material-icons">link</span> Copy Link
                  </button>
                  <div class="winner-select">
                    <button 
                      class:selected={map.winner === team1Name}
                      on:click={() => selectMapWinner(map, team1Name)}
                    >
                      {team1Name} Won
                    </button>
                    <button 
                      class:selected={map.winner === team2Name}
                      on:click={() => selectMapWinner(map, team2Name)}
                    >
                      {team2Name} Won
                    </button>
                  </div>
                </div>
              </div>
            {/if}
          </div>
        {/each}
      </div>
    {/each}
  </div>

  <div class="upcoming-map">
    <h2>Upcoming Map</h2>
    {#if getUpcomingMap()}
      <div class="map-card">
        <img src={getUpcomingMap().image} alt={getUpcomingMap().name} />
        <div class="map-info">
          <div class="map-name">{getUpcomingMap().name}</div>
          <div>{getUpcomingMap().selectedDifficulty}</div>
        </div>
      </div>
    {:else}
      <p>No upcoming maps</p>
    {/if}
  </div>

  <div class="map-pool">
    <h2>Map Pool</h2>
    <div class="map-grid">
      {#each mapPool as map}
        <!-- svelte-ignore a11y-no-static-element-interactions -->
        <div class="map-card" draggable="true" on:dragstart={(e) => onDragStart(e, map)}>
          <img src={map.image} alt={map.name} />
          <div class="map-info">
            <div class="map-name">{map.name}</div>
            <button on:click={() => removeMap(map)} class="remove-btn">
              <span class="material-icons">delete</span>
            </button>
          </div>
        </div>
      {/each}
    </div>
  </div>

  <div class="action-buttons">
    <button
      on:click={createMatchFlow}
      class="create-flow-btn"
      class:disabled={!isMatchFlowComplete}
      disabled={!isMatchFlowComplete}
    >
      <span class="material-icons">playlist_add_check</span>
      Create Match Flow
    </button>
    <button on:click={resetMatch} class="reset-btn">
      <span class="material-icons">refresh</span>
      Reset Match
    </button>
    <button on:click={copyMatchData} class="copy-data-btn">
      <span class="material-icons">content_copy</span>
      Copy Match Data
    </button>
  </div>
</main>

<style>
  :global(body) {
    font-family: 'Roboto', Arial, sans-serif;
    background-color: #121212;
    color: #ffffff;
    margin: 0;
    padding: 20px;
  }

  main {
    max-width: 1800px;
    margin: 0 auto;
    padding: 0 20px;
  }

  h1 {
    text-align: center;
    color: #bb86fc;
    font-size: 2.5rem;
    margin-bottom: 2rem;
  }

  h2 {
    color: #03dac6;
    font-size: 1.5rem;
    margin-bottom: 1rem;
  }

  .team-names {
    display: flex;
    justify-content: space-between;
    margin-bottom: 20px;
  }

  input {
    background-color: #2c2c2c;
    border: none;
    padding: 12px;
    color: #ffffff;
    border-radius: 4px;
    font-size: 1rem;
    transition: background-color 0.3s;
  }

  input:focus {
    background-color: #3c3c3c;
    outline: none;
  }

  .map-input {
    display: flex;
    margin-bottom: 20px;
  }

  .map-input input {
    flex-grow: 1;
    margin-right: 10px;
  }

  button {
    background-color: #bb86fc;
    color: #000000;
    border: none;
    padding: 12px 20px;
    cursor: pointer;
    display: flex;
    align-items: center;
    border-radius: 4px;
    transition: background-color 0.3s, transform 0.1s;
    font-size: 1rem;
  }

  button:hover {
    background-color: #3700b3;
    color: #ffffff;
    transform: translateY(-2px);
  }

  button:active {
    transform: translateY(0);
  }

  .columns {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 30px;
    margin-bottom: 30px;
  }

  .column {
    background-color: #2c2c2c;
    padding: 20px;
    border-radius: 8px;
    min-height: 400px;
    transition: background-color 0.3s;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  }

  .column.drag-over {
    background-color: #3700b3;
  }

  .ban {
    border: 2px solid #cf6679;
  }

  .pick {
    border: 2px solid #03dac6;
  }

  .map-card {
    background-color: #1f1f1f;
    margin-bottom: 15px;
    border-radius: 8px;
    overflow: hidden;
    transition: all 0.3s ease;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
  }

  .map-header {
    display: flex;
    align-items: center;
    padding: 12px;
    cursor: pointer;
  }

  .map-card img {
    width: 80px;
    height: 80px;
    object-fit: cover;
    border-radius: 4px;
    margin-right: 12px;
  }

  .map-info {
    flex-grow: 1;
  }

  .map-name {
    font-weight: bold;
    margin-bottom: 5px;
    font-size: 1.1rem;
  }

  select {
    background-color: #2c2c2c;
    color: #ffffff;
    border: none;
    padding: 6px;
    border-radius: 4px;
    font-size: 0.9rem;
  }

  .map-details {
    padding: 12px;
    border-top: 1px solid #3c3c3c;
  }

  .map-actions {
    display: flex;
    justify-content: space-between;
    margin-top: 10px;
  }

  .winner-select {
    display: flex;
    gap: 10px;
  }

  .winner-select button {
    padding: 8px 12px;
    font-size: 0.9rem;
  }

  .winner-select button.selected {
    background-color: #03dac6;
    color: #000000;
  }

  .upcoming-map {
    background-color: #2c2c2c;
    padding: 20px;
    border-radius: 8px;
    margin-bottom: 30px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  }

  .map-pool {
    background-color: #2c2c2c;
    padding: 20px;
    border-radius: 8px;
    margin-bottom: 30px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  }

  .map-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 20px;
  }

  .action-buttons {
    display: flex;
    justify-content: space-between;
    margin-top: 30px;
  }

  .create-flow-btn, .reset-btn, .copy-data-btn {
    flex: 0 1 30%;
    margin: 0;
  }

  .create-flow-btn {
    background-color: #03dac6;
  }

  .reset-btn {
    background-color: #cf6679;
  }

  .copy-data-btn {
    background-color: #bb86fc;
  }

  .disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .winner-announcement {
    background-color: #3700b3;
    color: #ffffff;
    padding: 15px;
    border-radius: 8px;
    text-align: center;
    margin-bottom: 20px;
    animation: pulse 2s infinite;
  }

  @keyframes pulse {
    0% {
      box-shadow: 0 0 0 0 rgba(187, 134, 252, 0.4);
    }
    70% {
      box-shadow: 0 0 0 10px rgba(187, 134, 252, 0);
    }
    100% {
      box-shadow: 0 0 0 0 rgba(187, 134, 252, 0);
    }
  }
</style>