<script setup>
// import Check from '@/icons/check.vue';
import userData from "@/api/users.json";
import modules from "@/api/modules.json";
import roles from "@/api/roles.json";
import moduleRoleData from "@/api/moduleRoleAccess.json";
import { onMounted, ref } from "vue";
import Loader from "@/components/loader.vue";
const db = ref(null);
const isloading = ref(false)
const items = ref([]);
const getUserRole = (id) => {
  const userRole = roles.find((role) => role.id == id);
  return userRole.name;
};

const shouldDisable = (moduleId, userId) => {
  const hasAccess = moduleRoleData.find(
    (moduleRole) =>
      moduleRole.module_id === moduleId &&
      moduleRole.accessibleUser.includes(userId)
  );
  if (!hasAccess) return true;
  return false;
};

const isChecked = (moduleId, userId) => {
  return items.value.some((item) => {
    return item.module_id === moduleId && item.isChecked.includes(userId);
  });
};

const handleChange = (e, data) => {
  if (!db.value) return;
  if (e.target.checked) {
    addData(data);
    return;
  }
  deleteData(data);
};

const deleteData = async (data) => {
  console.log("I WILL DELETE");
  const transaction = await db.value.transaction("items", "readwrite");
  const store = transaction.objectStore("items");
  const request = store.openCursor();
  let found = false;
  request.onsuccess = (e) => {
    const cursor = e.target.result;
    if (cursor) {
      if (cursor.value.module_id === data.moduleId) {
        const originalLength = cursor.value.isChecked.length;
        cursor.value.isChecked = cursor.value.isChecked.filter(
          (userId) => userId !== data.userId
        );
        if (cursor.value.isChecked.length !== originalLength) {
          found = true;
          const updateRequest = cursor.update(cursor.value);
          updateRequest.onsuccess = () => {
            console.log("Record updated successfully:", cursor.value);
          };
          updateRequest.onerror = (err) => {
            console.error("Failed to update record:", err);
          };
        }
      }
      cursor.continue();
    } else {
      if (!found) {
        console.log("No record found to delete or modify.");
      }
    }
  };

  transaction.oncomplete = () => {
    if (found) {
      console.log("SUCCESS DATA DELETED");
    } else {
      console.log("No matching data found to delete.");
    }
  };
};
const initDB = () => {
  const request = indexedDB.open("roleAccess", 1);
  request.onupgradeneeded = (event) => {
    const dbInstance = event.target.result;
    if (!dbInstance.objectStoreNames.contains("items")) {
      dbInstance.createObjectStore("items", {
        keyPath: "id",
        autoIncrement: true,
      });
    }
  };

  request.onsuccess = (event) => {
    db.value = event.target.result;
    fetchItems();
  };

  request.onerror = () => {};
};
const addData = async (data) => {
  const transaction = await db.value.transaction("items", "readwrite");
  const store = transaction.objectStore("items");
  const request = store.openCursor();
  let found = false;

  request.onsuccess = (e) => {
    const cursor = e.target.result;
    if (cursor) {
      if (cursor.value.module_id === data.moduleId) {
        found = true;
        cursor.value.isChecked.push(data.userId);
        const updateRequest = cursor.update(cursor.value);
        updateRequest.onsuccess = () => {
          console.log("Record updated successfully:", cursor.value);
        };
        updateRequest.onerror = (err) => {
          console.error("Failed to update record:", err);
        };
      }
      cursor.continue();
    } else {
      if (!found) {
        store.put({
          isChecked: [data.userId],
          module_id: data.moduleId,
        });
        console.log("New record added with module_id:", data.moduleId);
      }
    }
  };

  transaction.oncomplete = () => {
    console.log("SUCCESS DATA ADDED");
  };
};
const fetchItems = async () => {
  isloading.value = true
  if (!db.value) return;
  const transaction = db.value.transaction("items", "readonly");
  const store = transaction.objectStore("items");
  const request = store.getAll();
  request.onsuccess = () => {
    isloading.value = false
    items.value = request.result;
  };
};

onMounted(() => {
  initDB();
});
</script>

<template>
  <main class="main">
    <Loader v-if="isloading" />
    <table v-else>
      <thead>
        <td>Roles</td>
        <td v-for="module in modules" :key="module.id">
          {{ module.name }}
        </td>
      </thead>
      <tbody>
        <tr v-for="user in userData" :key="user.id">
          <td>
            <div class="first-td">
              <span class="name">
                {{ user.user_name }}
              </span>
              <span class="role">
                {{ getUserRole(user.role_id) }}
              </span>
            </div>
          </td>
          <td v-for="(m, index) in modules" :key="index">
            <label class="checkbox">
              <input
                type="checkbox"
                @change="
                  handleChange($event, { moduleId: m.id, userId: user.user_id })
                "
                class="chb chb-3"
                :disabled="shouldDisable(m.id, user.user_id)"
                :checked="
                  isChecked(m.id, user.user_id) &&
                  !shouldDisable(m.id, user.user_id)
                "
              />
              <span class="checkmark"></span>
            </label>
          </td>
        </tr>
      </tbody>
    </table>
  </main>
</template>

<style scoped>
.main {
  height: 100vh;
  width: 100%;
}

table {
  width: 100%;
  background: #fff;
  max-width: 1240px;
  margin: 0 auto;
  position: relative;
  border-collapse: collapse;
}
th,
td {
  padding: 15px 20px;
  text-align: left;
  min-width: 185px;
  max-width: 215px;
  width: auto;
  text-align: center;
}
thead {
  position: sticky;
  top: 0;
  background: #fff;
  z-index: 5;
}
th:first-child,
td:first-child {
  position: sticky;
  left: 0;
  z-index: 1;
  background-color: #f5f5f5;
  outline: 1px solid #dedede;
  min-width: 285px;
  max-width: 315px;
  width: auto;
}
thead td {
  background-color: #f5f5f5;
  font-weight: 900;
}
td:first-child {
  text-align: left;
}
td {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  text-align: center;
  border-right: 1px solid #dedede;
}
.name,
.role {
  display: block;
  font-size: 16px;
}
.name {
  font-weight: 900;
}
input[type=checkbox] {
  position: relative;
  border: 2px solid #000;
  border-radius: 5px;
  background: none;
  cursor: pointer;
  line-height: 0;
  margin: 0 .6em 0 0;
  outline: 0;
  padding: 0 !important;
  vertical-align: text-top;
  height: 25px;
  width: 25px;
  -webkit-appearance: none;
  opacity: .5;
}

input[type=checkbox]:hover {
  opacity: 1;
}

input[type=checkbox]:checked {
  background-color: #000;
  opacity: 1;
}

input[type=checkbox]:disabled {
  background-color: #050505;
  border: 2px solid #050505;
  cursor: not-allowed;
  opacity: 0.5;
}

input[type=checkbox]:checked::before {
  content: '';
  position: absolute;
  right: 50%;
  top: 50%;
  width: 5px;
  height: 16px;
  border: solid #FFF;
  border-width: 0 2px 3px 0;
  margin: -2px 1px 0 -1px;
  transform: rotate(45deg) translate(-50%, -50%);
  z-index: 2;
}

input[type=checkbox]:disabled::before {
  content: none;
}

</style>
