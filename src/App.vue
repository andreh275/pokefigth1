<template>
  <div id="app" class="background">
    <div v-if="!battleStarted" class="menu">
      <h1 class="poke-title">
        <span class="poke-title-red">Poke</span>
        <span class="poke-title-yellow">Batallas</span>
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
          />
        </div>

        <div>
          <label>Seleccionar Estadística 1:</label>
          <q-btn-dropdown
            color="primary"
            :label="getDropdownLabel(statsSelection1, 'Jugador 1')"
          >
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

        <div>
          <label>Seleccionar Estadística 2:</label>
          <q-btn-dropdown
            color="secondary"
            :label="getDropdownLabel(statsSelection2, 'Jugador 2')"
          >
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

        <button @click="startBattle">Confirmar Rondas</button>
      </div>
    </div>

    <!-- Campo de batalla -->
    <div v-if="battleStarted" class="battle-field">
      <h1>Batalla Pokémon - Ronda {{ currentRound + 1 }} de {{ rounds }}</h1>
      <div class="battle-container">

        <div class="pokemon-card" :style="{ backgroundColor: getTypeColor(player1Pokemon.types[0].type.name) }">
          <p class="player-name">Jugador 1</p> <!-- Nombre del jugador -->
          <img :src="player1Pokemon.sprites.front_default" alt="Pokémon" class="battle-sprite" />
          <p class="pokemon-name">{{ capitalize(player1Pokemon.name) }}</p>
          <p v-for="stat in combinedStats" :key="stat" v-if="roundResult">
            {{ capitalize(stat) }}: {{ getStatValue(player1Pokemon, stat) }}
          </p>
        </div>

        <div class="versus">VS</div>

        <div class="pokemon-card" :style="{ backgroundColor: getTypeColor(player2Pokemon.types[0].type.name) }">
          <p class="player-name">Jugador 2</p> <!-- Nombre del jugador -->
          <img :src="player2Pokemon.sprites.front_default" alt="Pokémon" class="battle-sprite" />
          <p class="pokemon-name">{{ capitalize(player2Pokemon.name) }}</p>
          <p v-for="stat in combinedStats" :key="stat" v-if="roundResult">
            {{ capitalize(stat) }}: {{ getStatValue(player2Pokemon, stat) }}
          </p>
        </div>
      </div>

      <div class="battle-actions">
        <button @click="combatir" v-if="!roundResult">Combatir</button>
      </div>

      <div v-if="roundResult" class="winner-message">
        <span class="gan" v-if="winner.name !== 'Empate'">
          ¡Ganador de la ronda: {{ capitalize(winner.name) }} ({{ winner.player }})!
        </span>
        <span v-else>¡Empate!</span>
      </div>

      <!-- Mostrar estadísticas de victorias y derrotas -->
      <div v-if="roundResult" class="scoreboard">
        <p class="juga">Victorias Jugador 1: {{ player1Wins }}</p>
        <p class="juga">Victorias Jugador 2: {{ player2Wins }}</p>
        <p class="juga">Derrotas Jugador 1: {{ currentRound + 1 - player1Wins }}</p>
        <p class="juga">Derrotas Jugador 2: {{ currentRound + 1 - player2Wins }}</p>
      </div>

      <div v-if="currentRound < rounds - 1 && roundResult">
        <button @click="nextRound" class="next-round-button">Siguiente Ronda</button>
      </div>

      <div v-else-if="roundResult">
        <h2 class="torneofinish">¡El Torneo ha terminado!</h2>
        <p class="jugador1a">Jugador 1 victorias: {{ player1Wins }}</p>
        <p class="jugador2a">Jugador 2 victorias: {{ player2Wins }}</p>
        <p class="Ganador" v-if="player1Wins > player2Wins">¡El ganador total es Jugador 1!</p>
        <p class="Ganador" v-if="player2Wins > player1Wins">¡El ganador total es Jugador 2!</p>
        <p class="Ganador" v-if="player1Wins === player2Wins">¡Empate!</p>
        <button @click="reset" class="reset-button">Reiniciar</button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue';
import axios from 'axios';
import Swal from 'sweetalert2';

export default {
  setup() {
    const rounds = ref(1);
    const statsList = ['hp', 'attack', 'defense', 'special-attack', 'special-defense', 'speed'];
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
    const winner = ref({ name: null, player: null }); // Almacena el nombre del Pokémon y del jugador
    const battleStarted = ref(false);
    const roundResult = ref(false);
    
    const fetchAllPokemons = async () => {
      try {
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
        Swal.fire({
          title: "Selecciona Estadísticas",
          text: "Debes seleccionar al menos una estadística para continuar.",
          icon: "warning",
          confirmButtonText: "Aceptar",
        });
        return;
      }
      combinedStats.value = [...statsSelection1.value, ...statsSelection2.value];
      rounds.value = Math.min(rounds.value, 10);  // Limitar rondas a un máximo de 10
      player1Pokemons.value = getRandomPokemons();
      player2Pokemons.value = getRandomPokemons();
      player1Pokemon.value = player1Pokemons.value[0];
      player2Pokemon.value = player2Pokemons.value[0];
      battleStarted.value = true;
    };

    const combatir = () => {
      if (!roundResult.value) {
        determineWinner();
        roundResult.value = true;
      }
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
        winner.value = { name: player1Pokemon.value.name, player: 'Jugador 1' };
        player1Wins.value++;
      } else if (player1Total < player2Total) {
        winner.value = { name: player2Pokemon.value.name, player: 'Jugador 2' };
        player2Wins.value++;
      } else {
        winner.value = { name: 'Empate', player: null };
      }
    };

    const nextRound = () => {
      currentRound.value++;
      if (currentRound.value < rounds.value) {
        player1Pokemon.value = player1Pokemons.value[currentRound.value];
        player2Pokemon.value = player2Pokemons.value[currentRound.value];
        roundResult.value = false;
        winner.value = { name: null, player: null };
      } else {
        roundResult.value = true;  // Para mostrar mensaje de fin de torneo
      }
    };

    const reset = () => {
      rounds.value = 1;
      statsSelection1.value = [];
      statsSelection2.value = [];
      player1Wins.value = 0;
      player2Wins.value = 0;
      currentRound.value = 0;
      winner.value = { name: null, player: null };
      battleStarted.value = false;
      roundResult.value = false;
      player1Pokemon.value = null;
      player2Pokemon.value = null;
    };

    const toggleStat = (stat, selection) => {
      if (selection === 'statsSelection1') {
        const index = statsSelection1.value.indexOf(stat);
        if (index > -1) {
          statsSelection1.value.splice(index, 1);
        } else {
          statsSelection1.value.push(stat);
        }
      } else {
        const index = statsSelection2.value.indexOf(stat);
        if (index > -1) {
          statsSelection2.value.splice(index, 1);
        } else {
          statsSelection2.value.push(stat);
        }
      }
    };

    const getDropdownLabel = (selection, player) => {
      return selection.length > 0
        ? `${player}: ${selection.join(', ')}`
        : `${player}: Selecciona Estadística`;
    };

    const getTypeColor = (type) => {
      const typeColors = {
        grass: '#78C850',
        fire: '#F08030',
        water: '#6890F0',
        electric: '#F8D030',
        ice: '#98D8D8',
        fighting: '#C03028',
        poison: '#A040A0',
        ground: '#E0C068',
        flying: '#A890F0',
        psychic: '#F85888',
        bug: '#A8B820',
        rock: '#B8A038',
        ghost: '#705898',
        dragon: '#7038F8',
        dark: '#705848',
        steel: '#B8B8D0',
        fairy: '#F0B6C1',
      };
      return typeColors[type] || '#FFFFFF';
    };

    const capitalize = (str) => {
      return str.charAt(0).toUpperCase() + str.slice(1);
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
      roundResult,
      startBattle,
      combatir,
      nextRound,
      reset,
      toggleStat,
      getDropdownLabel,
      getStatValue,
      getTypeColor,
      capitalize,
    };
  },
};
</script>

<style scoped>


.menu {
  text-align: center;
}

.menu-options {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.pokemon-card {
  width: 200px;
  height: 300px;
  border-radius: 10px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin: 10px;
}

.battle-container {
  display: flex;
  align-items: center;
}

.versus {
  color: #007e1f;
  font-size: xxx-large;
}

.battle-field {
  text-align: center;
}

.battle-sprite {
  width: 100px;
  height: 100px;
}

.winner-message {

  margin-top: 20px;
  font-size: 1.5rem;
}

.next-round-button,
.reset-button {
  margin-top: 20px;
  padding: 10px 20px;
}

.poke-title {
  font-size: 2rem;
}

.poke-title-red {
  color: red;
}

.poke-title-yellow {
  color: yellow;
}

.player-name {
  font-weight: bold;
  font-size: 1.2rem;
  margin-bottom: 10px; /* Espaciado entre el nombre del jugador y el Pokémon */
}

.scoreboard {
  margin-top: 20px;
  font-size: 1rem; /* Tamaño de fuente más pequeño para el marcador */
}

.menu {
  text-align: center;
}

.menu-options {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.pokemon-card {
  width: 200px;
  height: 300px;
  border-radius: 10px;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  margin: 10px;
}

.battle-container {
  display: flex;
  align-items: center;
}


.battle-field {
  text-align: center;
}

.battle-sprite {
  width: 100px;
  height: 100px;
}

.winner-message {
  margin-top: 20px;
  font-size: 1.5rem;
}

.next-round-button,
.reset-button {
  margin-top: 20px;
  padding: 10px 20px;
}

.poke-title {
  font-size: 2rem;
}

.poke-title-red {
  color: red;
}

.poke-title-yellow {
  color: yellow;
}

.player-name {
  font-weight: bold;
  font-size: 1.2rem;
  margin-bottom: 10px; /* Espaciado entre el nombre del jugador y el Pokémon */
}

@keyframes ganando {
0% {
  transform: scale(1);
  opacity: 1;
}
50% {
  transform: scale(1.2);
  opacity: 0.8;
}
100% {
  transform: scale(1);
  opacity: 1;
}
}

@keyframes resaltandoJugador {
0% {
  text-shadow: 0 0 5px #000000, 0 0 10px #000000, 0 0 15px #ff0000;
}
50% {
  text-shadow: 0 0 10px #000000, 0 0 20px #000000, 0 0 30px  #ff0000;
}
100% {
  text-shadow: 0 0 5px #000000, 0 0 10px #000000, 0 0 15px  #ff0000;
}
}

@keyframes torneoFinalizado {
  0% {
  transform: scale(1);
  opacity: 1;
}
50% {
  transform: scale(1.2);
  opacity: 0.8;
}
100% {
  transform: scale(1);
  opacity: 1;
}
}

.Ganador {
  font-size: x-large;
color: rgb(28, 58, 59);
animation: ganando 2s ease-in-out infinite;
}

.jugador1a,
.jugador2a {
font-size: x-large;
color: #ff0000;
animation: resaltandoJugador 1.5s ease-in-out infinite;
}

.torneofinish {
color: #e34c26;
animation: torneoFinalizado 2.5s ease-in-out infinite;
}

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



.winner-message {
  color: #000;
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
margin: 0;
padding: 0;
height: 0vh;
}


#app {
  background-image: url('https://i.pinimg.com/originals/7b/e7/de/7be7deba83eea0bfd0e0822a3419539a.gif'); 
  background-size: cover; 
  background-repeat: no-repeat; 
  background-attachment: fixed;
  background-position: center;
  height: 120vh;
  width: 100%;
  max-width: 1900px;
  margin: auto;
  background-color: transparent; 
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
  background-color: rgba(247, 214, 244, 0.2);
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
  background-color: #90d6c7;
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

.winner-message[data-v-7a7a37b1] {
  margin-top: 40px;
  font-size: 1.5rem;
  font-weight: bold;
  color: #ffffff;
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
  color: #fff;
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
  background: rgba(0, 0, 0, 0.2);
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

.selected {
  background-color: #d3d3d3;
}

.pokemon-card p,
.pokemon-card .pokemon-name {
  color: #fff;
  text-shadow: 1px 1px 2px #000;
}


@keyframes winnerAnimation {
  0% { opacity: 1; }
  50% { opacity: 0.8; }
  100% { opacity: 1; }
}

@media (max-width: 600px) {
  .battle-container {
    flex-direction: column;
  }

  .pokemon-card {
    margin: 10px 0;
  }

  .container {
    padding: 10px;
  }

  .battle-area {
    width: 100%;
  }
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
  color: #fff;
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
  background: rgba(0, 0, 0, 0.2); 
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


.pokemon-card p,
.pokemon-card .pokemon-name {
  color: #fff;
  text-shadow: 1px 1px 2px #000;
}

@media (max-width: 600px) {
  .battle-container {
    flex-direction: column;
  }

  .pokemon-card {
    margin: 10px 0;
  }
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
.gan{
  color: #c22f1f;
}
.juga{
  font-size: X-large;
  color: #306300;
}
</style> 
