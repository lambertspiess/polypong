<template>
  <q-page padding>
    <q-card>
      <q-card-section>
        <q-input label="Change your username" v-model="newName"></q-input>
        <q-btn @click="changeName(newName)">change name</q-btn>
      </q-card-section>
    </q-card><br>

    <q-card>
      <q-card-section>
        <q-card-section v-if="
          $auth.user.isTwoFactorAuthenticationEnabled === true
        ">
          <h6> 2fa is required </h6>
          <q-btn @click="turnOff2fa()">turn off 2fa</q-btn><br>
        </q-card-section>
        <q-card-section v-else>
          <h6> 2fa is not required </h6>
          <q-btn @click="requestQrCode()">request QrCode</q-btn><br>
          <q-img :src=qrCode.imageBytes width="50%"></q-img><br>
          <q-input
            label="Scan with Google Authenticator and enter code here"
            v-model="qrCode.decoded"
          >
            <template v-slot:append>
              <q-btn @click="activate2fa(qrCode.decoded)">activate 2FA</q-btn><br>
            </template>
          </q-input><br>
        </q-card-section>
      </q-card-section>
    </q-card><br>

    <q-card>
      <q-card-section class="column flex-center">
        <h6 class="self-start">current avatar :</h6>
        <q-img
          :src="$auth.user.avatar"
          style="max-width: 50%; max-height: 50%; border-radius: 15px;"
        />
        <q-uploader
          dark accept="image/*" :factory="factoryFn" field-name="avatar"
          @finish="$auth.fetchUser()"
        />
      </q-card-section>
    </q-card><br>

    <q-card>
      <q-card-section class="column flex-center">
          <q-btn @click="logOut()" color="red">LOGOUT</q-btn><br>
      </q-card-section>
    </q-card><br>

  </q-page>

</template>

<script lang="ts" setup>
import { useAuthStore } from 'src/stores/auth.store';
import { Notify } from 'quasar';
import { ref } from 'vue';
import { useRouter } from 'vue-router';

const $auth = useAuthStore();
const router = useRouter();

// name change
const newName = ref('');
const changeName = async (name) => {
  try {
    await $auth.updateUser({ name });
  } catch ({ response, body }) {
    // Notify.create({
    //  type: 'negative',
    //  message: (body as any).message,
    // });
  }
};

// 2fa
const qrCode = ref({
  requested: false,
  imageBytes: '',
  decoded: '',
});
async function requestQrCode() {
  const res: any = await $auth.requestQrCode();
  qrCode.value.requested = true;
  qrCode.value.imageBytes = res.qrAsDataUrl;
}
async function turnOff2fa() {
  await $auth.updateUser({ isTwoFactorAuthenticationEnabled: false });
  qrCode.value.imageBytes = ''; qrCode.value.decoded = '';
}
async function activate2fa(value: any) {
  try {
    await $auth.activate2fa(value);
    router.push({ name: '2fa' });
  } catch ({ response, body }) {
    // Notify.create({
    //  type: 'negative',
    //  message: (body as any).message,
    // });
  }
}

function factoryFn(file: any): Promise<any> {
  return new Promise((resolve, reject) => {
    // Retrieve JWT token from your store.
    resolve({
      url: '/api/user/setAvatar',
      method: 'POST',
      headers: [
        { name: 'Authorization', value: `Bearer ${localStorage.getItem('token')}` },
      ],
    });
  });
}

// logout
function logOut(): void {
  $auth.killws();
  localStorage.removeItem('token');
  router.push({ name: 'login' });
}

</script>
