<template>
  <div id="app">
    <div v-if="!battleStarted" class="menu">
      <h1 class="poke-title">
        <span class="poke-title-red">Poke</span><span class="poke-title-yellow">Batallas</span>
      </h1>
      <div class="menu-options">
        <div>
  <label for="rounds">Número de rondas (1-10):</label>
  <input
    v-model.number="rounds"
    id="rounds"
    type="number"
    min="1"
    max="10"
    @input="validateRounds" 
  />
</div>

        <!-- Selección de estadísticas - Jugador 1 -->
        <div>
          <label>Seleccionar Estadísticas (Jugador 1):</label>
          <q-btn-dropdown color="primary" :label="getDropdownLabel(statsSelection1, 'Jugador 1')">
            <q-list>
              <q-item
                v-for="stat in statsList"
                :key="stat"
                clickable
                :class="{ selected: statsSelection1.includes(stat) }"
                @click="toggleStat(stat, 'statsSelection1')"
              >
                <q-item-section>
                  <q-item-label>{{ stat }}</q-item-label>
                </q-item-section>
              </q-item>
            </q-list>
          </q-btn-dropdown>
        </div>

        <!-- Selección de estadísticas - Jugador 2 -->
        <div>
          <label>Seleccionar Estadísticas (Jugador 2):</label>
          <q-btn-dropdown color="secondary" :label="getDropdownLabel(statsSelection2, 'Jugador 2')">
            <q-list>
              <q-item
                v-for="stat in statsList"
                :key="stat"
                clickable
                :class="{ selected: statsSelection2.includes(stat) }"
                @click="toggleStat(stat, 'statsSelection2')"
              >
                <q-item-section>
                  <q-item-label>{{ stat }}</q-item-label>
                </q-item-section>
              </q-item>
            </q-list>
          </q-btn-dropdown>
        </div>

        <!-- Botón para confirmar rondas y comenzar la batalla -->
        <button @click="startBattle">Confirmar Rondas</button>
      </div>
    </div>

    <!-- Campo de batalla -->
    <div v-if="battleStarted" class="battle-field">
      <h1>Batalla Pokémon - Ronda {{ currentRound + 1 }} de {{ rounds }}</h1>
      <div class="battle-container">
        <!-- Tarjeta del Pokémon del Jugador 1 -->
        <div
          class="pokemon-card"
          :style="{ backgroundColor: getTypeColor(player1Pokemon.types[0].type.name) }"
        >
          <img :src="player1Pokemon.sprites.front_default" alt="Pokémon" class="battle-sprite" />
          <p class="pokemon-name">{{ capitalize(player1Pokemon.name) }}</p>
          <p v-for="stat in combinedStats" :key="stat">{{ capitalize(stat) }}: {{ getStatValue(player1Pokemon, stat) }}</p>
        </div>

        <div class="versus">VS</div>

        <!-- Tarjeta del Pokémon del Jugador 2 -->
        <div
          class="pokemon-card"
          :style="{ backgroundColor: getTypeColor(player2Pokemon.types[0].type.name) }"
        >
          <img :src="player2Pokemon.sprites.front_default" alt="Pokémon" class="battle-sprite" />
          <p class="pokemon-name">{{ capitalize(player2Pokemon.name) }}</p>
          <p v-for="stat in combinedStats" :key="stat">{{ capitalize(stat) }}: {{ getStatValue(player2Pokemon, stat) }}</p>
        </div>
      </div>

      <div v-if="winner" class="winner-message">
        <span v-if="winner.name !== 'Empate'">¡Ganador de la ronda: {{ capitalize(winner.name) }}!</span>
        <span v-else>¡Empate!</span>
      </div>

      <div v-if="currentRound < rounds - 1">
        <button @click="nextRound" class="next-round-button">Siguiente Ronda</button>
      </div>
      <div v-else>
        <h2>¡El Torneo ha terminado!</h2>
        <p>Jugador 1 victorias: {{ player1Wins }}</p>
        <p>Jugador 2 victorias: {{ player2Wins }}</p>
        <p v-if="player1Wins > player2Wins">¡El ganador total es Jugador 1!</p>
        <p v-if="player2Wins > player1Wins">¡El ganador total es Jugador 2!</p>
        <p v-if="player1Wins === player2Wins">¡Empate!</p>
        <button @click="reset" class="reset-button">Reiniciar</button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import axios from 'axios';

export default {
  setup() {
    const rounds = ref(1);
    const statsList = [
      'hp',
      'attack',
      'defense',
      'special-attack',
      'special-defense',
      'speed',
    ];

    const statsSelection1 = ref([]);
    const statsSelection2 = ref([]);
    const combinedStats = ref([]);

    const allPokemons = ref([]);
    const player1Pokemons = ref([]);
    const player2Pokemons = ref([]);
    const player1Pokemon = ref(null);
    const player2Pokemon = ref(null);
    const player1Wins = ref(0);
    const player2Wins = ref(0);
    const currentRound = ref(0);
    const winner = ref(null);
    const battleStarted = ref(false);

    const fetchAllPokemons = async () => {
      try {
        // Limitar a los primeros 151 Pokémon para mejorar el rendimiento
        const promises = Array.from({ length: 151 }, (_, i) =>
          axios.get(`https://pokeapi.co/api/v2/pokemon/${i + 1}`)
        );
        const responses = await Promise.all(promises);
        allPokemons.value = responses.map((response) => response.data);
      } catch (error) {
        console.error(error);
      }
    };

    const startBattle = () => {
      if (statsSelection1.value.length === 0 && statsSelection2.value.length === 0) {
        alert('Selecciona al menos una estadística para la batalla.');
        return;
      }

      combinedStats.value = [...statsSelection1.value, ...statsSelection2.value];

      player1Pokemons.value = getRandomPokemons();
      player2Pokemons.value = getRandomPokemons();
      player1Pokemon.value = player1Pokemons.value[0];
      player2Pokemon.value = player2Pokemons.value[0];
      battleStarted.value = true;
      determineWinner();
    };

    const getRandomPokemons = () => {
      const shuffledPokemons = [...allPokemons.value].sort(() => 0.5 - Math.random());
      return shuffledPokemons.slice(0, rounds.value);
    };

    const getStatValue = (pokemon, stat) => {
      const statObject = pokemon.stats.find((s) => s.stat.name === stat);
      return statObject ? statObject.base_stat : 0;
    };

    const determineWinner = () => {
      let player1Total = 0;
      let player2Total = 0;

      combinedStats.value.forEach((stat) => {
        player1Total += getStatValue(player1Pokemon.value, stat);
        player2Total += getStatValue(player2Pokemon.value, stat);
      });

      if (player1Total > player2Total) {
        winner.value = player1Pokemon.value;
        player1Wins.value++;
      } else if (player2Total > player1Total) {
        winner.value = player2Pokemon.value;
        player2Wins.value++;
      } else {
        winner.value = { name: 'Empate' };
      }
    };
    const nextRound = () => {
  // Comprobar si se ha determinado un ganador
  if (winner.value === null) {
    determineWinner(); // Asegurarse de que tenemos un ganador para la ronda actual
  }

  // Incrementar la ronda actual
  currentRound.value++;

  // Si hay más rondas por jugar
  if (currentRound.value < rounds.value) {
    // Actualizar Pokémon para la siguiente ronda
    if (winner.value && winner.value !== 'Empate') {
      // El Pokémon que ganó se queda, y el otro se reemplaza
      if (winner.value === player1Pokemon.value) {
        player2Pokemons.value.shift(); // Sacar al Pokémon perdedor de Jugador 2
      } else {
        player1Pokemons.value.shift(); // Sacar al Pokémon perdedor de Jugador 1
      }
    } else {
      // En caso de empate, simplemente se puede mantener ambos Pokémon
      player1Pokemon.value = player1Pokemons.value[0] || null;
      player2Pokemon.value = player2Pokemons.value[0] || null;
    }

    // Restablecer el ganador para la próxima ronda
    winner.value = null;

    // Actualizar los Pokémon para la próxima ronda
    player1Pokemon.value = player1Pokemons.value[0] || null;
    player2Pokemon.value = player2Pokemons.value[0] || null;

    // Determinar el ganador de la nueva ronda
    if (player1Pokemon.value && player2Pokemon.value) {
      determineWinner();
    }
  } else {
    // Si no hay más rondas, mostrar el resultado final
    showFinalResults();
  }
};

const showFinalResults = () => {
  // Aquí puedes agregar lógica para mostrar los resultados finales
  alert(`¡El Torneo ha terminado! Jugador 1 victorias: ${player1Wins.value}, Jugador 2 victorias: ${player2Wins.value}`);
};



    const reset = () => {
      player1Pokemons.value = [];
      player2Pokemons.value = [];
      player1Wins.value = 0;
      player2Wins.value = 0
      currentRound.value = 0;
      winner.value = null;
      battleStarted.value = false;
      statsSelection1.value = [];
      statsSelection2.value = [];
    };

  const getTypeColor = (type) => {
  const typeColors = {
    grass: '#78C850',
    fire: '#F08030',
    water: '#6890F0',
    bug: '#A8B820',
    normal: '#A8A878',
    poison: '#A040A0',
    electric: '#F8D030',
    ground: '#E0C068',
    fairy: '#EE99AC',
    fighting: '#C03028',
    psychic: '#F85888',
    rock: '#B8A038',
    ice: '#98D8D8',
    ghost: '#705898',
    dragon: '#7038F8',
  };
  return typeColors[type] || '#FFFFFF'; // Color por defecto
};

    const capitalize = (string) => {
      return string.charAt(0).toUpperCase() + string.slice(1);
    };

    const getDropdownLabel = (selection, player) => {
      return selection.length > 0
        ? `${player}: ${selection.join(', ')}`
        : `${player}: Seleccionar estadísticas`;
    };

    const toggleStat = (stat, selectionRef) => {
      const selection = selectionRef === 'statsSelection1' ? statsSelection1 : statsSelection2;
      if (selection.value.includes(stat)) {
        selection.value = selection.value.filter((s) => s !== stat);
      } else {
        selection.value.push(stat);
      }
    };

    onMounted(fetchAllPokemons);

    return {
      rounds,
      statsList,
      statsSelection1,
      statsSelection2,
      combinedStats,
      player1Pokemon,
      player2Pokemon,
      player1Wins,
      player2Wins,
      currentRound,
      winner,
      battleStarted,
      startBattle,
      nextRound,
      reset,
      getStatValue,
      getTypeColor,
      capitalize,
      getDropdownLabel,
      toggleStat,
    };
  },
  
};

</script>

<style scoped>
.menu {
  text-align: center;
  margin: 20px;
}

.battle-field {
  text-align: center;
  margin: 20px;
}

.pokemon-card {
  display: inline-block;
  width: 200px;
  padding: 10px;
  border-radius: 10px;
  margin: 10px;
  color: white;
}

.battle-sprite {
  width: 100px;
  height: 100px;
}

.versus {
  font-size: 2rem;
  margin: 20px;
}

.winner-message {
  font-size: 1.5rem;
  margin: 20px;
}

.next-round-button, .reset-button {
  margin-top: 20px;
  padding: 10px 20px;
  font-size: 1rem;
}

.pokemon-sprite {
  width: 100%;
  max-width: 150px;
  height: auto;
}


body {
  background-image: url('https://i.pinimg.com/originals/94/69/f6/9469f63cb17b971a97b02a54c0e5a962.gif');
  background-size: cover;
  background-repeat: no-repeat;
  background-position: center;
  margin: 0;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}

#app {
  width: 90%;
  max-width: 1200px;
  margin: auto;
  background-color: transparent; /* Fondo semitransparente */
  padding: 20px;
}

.menu {
  text-align: center;
}

.poke-title {
  font-size: 3rem;
  font-weight: bold;
  text-shadow: 3px 3px 0px #f7d6f4, -1px -1px 0px #ffcb05;
}

.poke-title-red {
  color: #ff0000;
}

.poke-title-yellow {
  color: #ffcb05;
}

.menu-options {
  background-color: #f7d6f4;
  padding: 20px;
  border-radius: 20px;
  display: inline-block;
  margin-top: 20px;
}

.menu-options > div {
  margin-bottom: 15px;
}

label {
  display: block;
  margin-bottom: 5px;
  font-weight: bold;
}

input {
  width: 100%;
  padding: 8px;
  border-radius: 8px;
  border: 1px solid #ccc;
}

button {
  background-color: #e34c26;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.3s ease;
}

button:hover {
  background-color: #c22f1f;
}

.battle-field {
  text-align: center;
  padding: 20px;
  border-radius: 20px;
}

.battle-container {
  display: flex;
  justify-content: space-around;
  align-items: center;
  margin-top: 30px;
  flex-wrap: wrap;
}

.winner-message {
  margin-top: 40px;
  font-size: 1.5rem;
  font-weight: bold;
  color: #a17e1e;
}

.pokemon-card {
  background-color: #ffffff;
  border: 2px solid #ddd;
  border-radius: 15px;
  margin: 10px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  text-align: center;
  padding: 20px;
  width: 250px;
  color: #fff; /* Texto en blanco para mejor legibilidad */
  position: relative;
}

.pokemon-card::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  border-radius: 15px;
  background: rgba(0, 0, 0, 0.2); /* Superposición oscura para resaltar el texto */
}

.pokemon-card img {
  width: 120px;
  height: 120px;
  object-fit: contain;
  z-index: 1;
}

.pokemon-name {
  font-size: 1.5rem;
  margin-top: 10px;
  z-index: 1;
  position: relative;
}

.pokemon-card p {
  z-index: 1;
  position: relative;
  margin: 5px 0;
}

.versus {
  font-size: 2rem;
  margin: 20px;
  z-index: 1;
}

.reset-button,
.next-round-button {
  background-color: #6f8a85;
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.3s ease;
  margin-top: 20px;
}

.reset-button:hover,
.next-round-button:hover {
  background-color: #5e7a70;
}

/* Estilos para las listas seleccionadas */
.selected {
  background-color: #d3d3d3;
}

/* Asegurar que el texto sobre fondos coloridos sea legible */
.pokemon-card p,
.pokemon-card .pokemon-name {
  color: #fff;
  text-shadow: 1px 1px 2px #000;
}

@media (max-width: 768px) {
  .pokemon-sprite {
    max-width: 100px;
  }

  .battlefield {
    min-height: 300px;
    max-height: 500px;
  }

  button {
    padding: 8px 16px;
    font-size: 0.9rem;
  }
  @media (max-width: 600px) {
  .container {
    padding: 10px;
  }

  .battle-area {
    width: 100%;
  }
}
}

</style>