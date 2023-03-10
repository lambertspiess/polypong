<style lang="scss">
  .card-grid {
    display: flex;
    flex-wrap: wrap;
    flex-grow: 1;
    gap: 10px;
    margin-bottom: 20px;

    > * {
      justify-content: center;
      flex-grow: 1;
    }
  }
</style>

<template>
  <q-page padding>
    <div class="card-grid">
      <MatchmakingCard
        :id="0"
        name="Matchmaking"
        subhead="Find a regular opponent"
        avatar="/matchmaking.png"
        :isCreated=true
        :madeMatches=$lobbies.madeMatches
        @joinMatchmake="joinMatchmake"
        @leaveMatchmake="leaveMatchmake"
        >
        Recent matches : {{$lobbies.madeMatches}}
      </MatchmakingCard>
      <LobbyCard
        :id="-1"
        name="Create"
        subhead="Create a new lobby"
        :isCreated=false
        @joinLobby="createLobby"
      >
        <q-input label="Lobby name" dense filled v-model="lobbyName"/>
      </LobbyCard>
      <LobbyCard
        v-for="lobby of $lobbies.getLobbies"
        :key="`lobby-${lobby.id}`"
        :id="lobby.id"
        :name="lobby.name || 'Unamed lobby'"
        :subhead="`${lobby.host.name}'s party`"
        :avatar="lobby.host.avatar"
        :isCreated=true
        :is-private="lobby.isPrivate"
        :is-started="lobby.isStarted"
        :is-full="lobby.players.length === lobby.playersMax"
        @joinLobby="router.push({ name: 'lobby', params: { id: lobby.id } })"
        @joinGame="router.push({ name: 'game', params: { id: lobby.id } })"
        @spectate="router.push({ name: 'game', params: { id: lobby.id } })"
      >
        <q-circular-progress
          show-value
          class="text-accent q-ma-md"
          track-color="grey-2"
          :color="lobby.isStarted ? 'accent' : 'primary'"
          :thickness=".2"
          size="50px"
          :indeterminate="!!lobby.isStarted"
          :value="lobby.players.length"
          :max="lobby.playersMax"
        >
          <span>{{ lobby.players.length }}</span>
          <q-icon name="fa fa-user-astronaut" />
          <q-tooltip>{{ lobby.players.length }} / {{ lobby.playersMax }} players</q-tooltip>
        </q-circular-progress>
        <q-circular-progress
          show-value
          class="text-primary q-ma-md"
          track-color="grey-2"
          :color="lobby.isStarted ? 'accent' : 'primary'"
          size="50px"
          :thickness=".2"
          :indeterminate="!!lobby.isStarted"
          :value="lobby.spectators.length"
          :max="lobby.spectatorsMax"
        >
          <span>{{ lobby.spectators.length }}</span>
          <q-icon name="fa fa-user-secret" />
        <q-tooltip>{{ lobby.spectators.length }} spectators</q-tooltip>
        </q-circular-progress>
      </LobbyCard>
    </div>
  </q-page>
</template>

<script lang="ts" setup>
import { useLobbiesStore } from 'src/stores/lobbies.store';
import { useAuthStore } from 'src/stores/auth.store';
import LobbyCard from 'src/components/LobbyCard.vue';
import PasswordInput from 'src/components/PasswordInput.vue';
import { useRouter } from 'vue-router';
import {
  onMounted,
  ref,
  defineComponent,
} from 'vue';
import MatchmakingCard from 'src/components/MatchmakingCard.vue';

defineComponent({
  components: {
    LobbyCard,
  },
});

const $lobbies = useLobbiesStore();
const { socket } = useAuthStore();
$lobbies.fetchLobbies();
const lobbyName = ref('');
const router = useRouter();

async function createLobby() {
  const newLobby = await $lobbies.createLobby(lobbyName.value);
  if (newLobby) {
    await $lobbies.fetchAndJoinLobby(newLobby.id);
    router.push({ name: 'lobby', params: { id: newLobby.id } });
  }
}

async function joinMatchmake() {
  $lobbies.joinMatchmake();
}
async function leaveMatchmake() {
  $lobbies.leaveMatchmake();
}

onMounted(() => {
  socket.on('update_lobbies', (evt) => {
    $lobbies.lobbies = evt;
  });
  socket.on('matchmake_done', (lobbyid) => {
    $lobbies.fetchAndJoinLobby(lobbyid).then(() => {
      router.push(`/lobby/${lobbyid}/game`);
    });
  });
});
</script>
