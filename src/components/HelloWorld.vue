<template>
  <div class="p-4">
    <!-- Xato matni -->
    <h1 class="text-danger" v-if="errorText">{{ errorText }}</h1>

    <!-- Loader -->
    <div
      v-if="loading"
      class="d-flex justify-content-center align-items-center"
    >
      <b-spinner label="Yuklanmoqda..."></b-spinner>
      <span class="ms-2">Ma'lumotlar yuklanmoqda...</span>
    </div>

    <div v-else>
      <!-- Saqlash tugmasi -->
      <div class="d-flex justify-content-end mb-3">
        <b-button variant="primary" @click="onSave">Saqlash</b-button>
      </div>

      <!-- Tabs -->
      <b-tabs>
        <!-- Oddiy userlar -->
        <b-tab title="Oddiy userlar">
          <b-form-input
            v-model="searchUsers"
            placeholder="User qidirish..."
            class="mb-2"
          ></b-form-input>

          <b-list-group>
            <b-list-group-item
              v-for="u in filteredNormalUsers"
              :key="u.user_id"
              class="d-flex justify-content-between align-items-center"
            >
              <div>
                <strong>{{ u.first_name }} {{ u.last_name }}</strong>
                <div class="text-muted" v-if="u.contacts?.length">
                  <div
                    v-for="c in u.contacts.filter(
                      (c) => c.contact_type === 'telegram_number'
                    )"
                    :key="c.contact_id"
                  >
                    Telegram number: +{{ c.contact_value }}
                  </div>
                </div>
              </div>
              <b-form-checkbox v-model="selectedForStaff" :value="u.user_id">
                Staff qil
              </b-form-checkbox>
            </b-list-group-item>
          </b-list-group>
        </b-tab>

        <!-- Stafflar -->
        <b-tab title="Stafflar">
          <b-table
            :items="staffUsers"
            :fields="['first_name', 'last_name', 'telegram_number', 'is_staff']"
            small
            bordered
            responsive
          >
            <template #cell(is_staff)="row">
              <b-form-checkbox v-model="row.item.is_staff">
                Staff
              </b-form-checkbox>
            </template>
          </b-table>
        </b-tab>
      </b-tabs>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from "vue";
import supabase from "../config/supabazaClient";

const errorText = ref("");
const loading = ref(false);

const searchUsers = ref("");
const normalUsers = ref([]); // oddiy userlar
const staffUsers = ref([]); // stafflar
const selectedForStaff = ref([]); // oddiy userlardan staff qilish uchun tanlanganlar

const queryParams = new URLSearchParams(window.location.search);
const org_id = queryParams.get("org_id") || null;

const Telegram = window.Telegram?.WebApp;

onMounted(async () => {
  if (!org_id) {
    errorText.value = "Organization ID topilmadi";
    return;
  }

  loading.value = true;

  // Oddiy userlar (staff bo‘lmaganlar)
  const { data: usersData, error: usersError } = await supabase.rpc(
    "get_users_with_cargos_by_org",
    { p_org_id: org_id }
  );

  // Stafflar
  const { data: staffData, error: staffError } = await supabase.rpc(
    "get_staff_users_by_org",
    { p_org_id: org_id }
  );

  if (usersError || staffError) {
    errorText.value =
      "Supabase xato: " +
      (usersError?.message || staffError?.message || "no error");
  } else {
    normalUsers.value = usersData.filter((el) => !el.isstaff) || [];
    staffUsers.value = staffData?.map((s) => ({ ...s, is_staff: true })) || [];
  }

  loading.value = false;
});

// Foydalanuvchilarni filtrlash (kontaktlar orqali ham)
const filteredNormalUsers = computed(() => {
  const term = searchUsers.value.trim().toLowerCase();
  if (!term) return normalUsers.value;

  return normalUsers.value.filter((u) => {
    const fields = [u.first_name, u.last_name, u.user_id].filter(Boolean);
    const contactValues = (u.contacts || [])
      .map((c) => c.contact_value)
      .filter(Boolean);

    const allFields = [...fields, ...contactValues];

    return allFields.some((f) => String(f).toLowerCase().includes(term));
  });
});

// Saqlash tugmasi bosilganda
const onSave = async () => {
  try {
    // Oddiy userlardan staff qilinishi kerak bo'lganlar
    const newStaff = selectedForStaff.value.map((userId) => ({
      user_id: userId,
      org_id,
      is_staff: true,
    }));

    // Stafflardan oddiy userga qaytarilishi kerak bo'lganlar
    const removedStaff = staffUsers.value
      .filter((s) => !s.is_staff)
      .map((s) => ({
        user_id: s.user_id,
        org_id,
        is_staff: false,
      }));

    // Telegram payload
    const queryId = Telegram.initDataUnsafe?.query_id;
    const payload = JSON.stringify({
      manage_staff: {
        newStaff,
        removedStaff,
        org_id,
      },
      queryId,
    });

    if (queryId) {
      fetch("https://telegram-bota-da4625226d63.herokuapp.com/web-data", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: payload,
      });
    } else {
      Telegram.sendData(payload);
    }

    alert("O‘zgarishlar yuborildi ✅");
  } catch (e) {
    console.error(e);
    errorText.value = "Saqlashda xato" + e;
  }
};
</script>
