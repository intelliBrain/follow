<template>
  <q-layout view="hHh Lpr lFr">
    <q-header class="bg-black row no-wrap">
      <q-toolbar>
        <q-btn flat round dense icon="menu" @click="drawer = !drawer" />
        <q-toolbar-title>Follow</q-toolbar-title>
        <div>ID: {{ ipfs_id.id }}</div>
        <q-space />
      </q-toolbar>

      <q-btn unelevated icon="person_add" @click="followPrompt = true" />
    </q-header>

    <q-drawer
      v-model="drawer"
      behavior="desktop"
      bordered
      show-if-above
      side="left"
    >
      <q-scroll-area class="fit">
        <q-list v-for="(menuItem, index) in menuList" :key="index">
          <q-item
            v-if="ipfs_id.id"
            v-ripple
            clickable
            :publisher="ipfs_id.id"
            :to="{ name: menuItem.route, params: { publisher: ipfs_id.id } }"
          >
            <q-item-section avatar>
              <q-icon :name="menuItem.icon" />
            </q-item-section>
            <q-item-section>{{ menuItem.label }}</q-item-section>
          </q-item>
        </q-list>

        <q-item>
          <q-item-section avatar>
            <q-icon name="style" />
          </q-item-section>
          <q-item-section>
            <q-toggle v-model="dark" color="green" />
          </q-item-section>
        </q-item>
      </q-scroll-area>
    </q-drawer>

    <!-- router view -->
    <q-page-container>
      <router-view
        v-if="ipfs_id.id"
        :key="$route.path"
        :publisher="ipfs_id.id"
        class="root-container"
        @show-unfollow-prompt="showUnfollowPrompt"
        @show-link-prompt="showLinkPrompt"
      />
    </q-page-container>

    <!-- share link modal -->
    <q-dialog v-model="shareLinkPrompt">
      <q-card>
        <q-card-section>
          <div class="text-h6">Link</div>
        </q-card-section>

        <q-card-section class="text-body2">{{ shareLink }}</q-card-section>

        <q-card-actions align="right">
          <q-btn v-close-popup flat label="OK" color="primary" />
        </q-card-actions>
      </q-card>
    </q-dialog>

    <!-- follow new id modal -->
    <q-dialog v-model="followPrompt" persistent>
      <q-card style="min-width: 350px">
        <q-card-section>
          <div class="text-h6">Enter an ID to follow</div>
        </q-card-section>

        <q-card-section class="q-pt-none">
          <q-input
            v-model="idToFollow"
            dense
            autofocus
            @keyup.enter="followPrompt = false"
          />
        </q-card-section>

        <q-card-actions align="right" class="text-primary">
          <q-btn v-close-popup flat label="Cancel" />
          <q-btn v-close-popup flat label="Add ID" @click="addFollowing()" />
        </q-card-actions>
      </q-card>
    </q-dialog>
    <!-- unfollow confirmation modal -->
    <q-dialog v-model="unfollowPrompt">
      <q-card>
        <q-card-section>
          <div class="text-body">Unfollow {{ idToUnollow }}?</div>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn v-close-popup flat label="Cancel" color="primary" />
          <q-btn
            v-close-popup
            flat
            label="Unfollow"
            color="primary"
            @click="removeFollowing()"
          />
        </q-card-actions>
      </q-card>
    </q-dialog>
    <!--  -->
  </q-layout>
</template>

<script>
import { defineComponent } from "vue";

const menuList = [
  {
    icon: "rss_feed",
    label: "Feed",
    route: "Feed",
  },
  {
    icon: "assignment_ind",
    label: "Profile",
    route: "Identity",
  },
  {
    icon: "settings",
    label: "Settings",
    route: "Settings",
  },
];

export default defineComponent({
  name: "App",
  data() {
    return {
      dark: true,
      drawer: false,
      followPrompt: false,
      idToFollow: "",
      idToUnollow: "",
      ipfs_id: {},
      menuList,
      publishInterval: null,
      refreshInterval: null,
      unfollowPrompt: false,
      shareLinkPrompt: false,
      shareLink: "",
    };
  },

  watch: {
    dark: {
      handler: function (after) {
        this.dark = after;
        this.$q.dark.set(this.dark);
      },
    },
  },
  created() {
    this.$q.dark.set(true);
  },
  beforeUnmount: function () {
    clearInterval(this.publishInterval);
    clearInterval(this.refreshInterval);
  },
  mounted: function () {
    // window.ipc.on("ipfs_id", (id) => {
    //   console.log("on.ipfs_id");
    //   this.ipfs_id = id;
    //   this.$store.commit("setIpfsId", id);
    // });
    // window.ipc.send("get-ipfs_id");
    window.ipc.invoke("get-ipfs_id").then((id) => {
      console.log("get-ipfs_id.then");
      console.log(id);
      this.ipfs_id = id;
      this.$store.commit("setIpfsId", id);
    });

    // window.ipc.on("identities", (identities) => {
    //   console.log("on.identities");
    //   console.log(identities);
    //   this.$store.commit("setIdentites", identities);
    // });
    // window.ipc.send("get-identities");
    window.ipc.invoke("get-identities").then((identities) => {
      console.log("get-identities.then");
      console.log(identities);
      this.$store.commit("setIdentites", identities);
    });

    window.ipc.send("publish-identity");
    window.ipc.send("update-following");

    this.publishInterval = setInterval(async function () {
      console.log("auto-publish...");
      window.ipc.send("publish-identity");
    }, 10 * 60 * 1000);
    this.refreshInterval = setInterval(async function () {
      console.log("refreshing identities...");
      window.ipc.send("update-following");
    }, 1 * 60 * 1000);
  },
  methods: {
    showUnfollowPrompt(id) {
      console.log(`App: showUnfollowPrompt(${id})`);
      this.idToUnollow = id;
      this.unfollowPrompt = true;
    },
    showLinkPrompt(link) {
      console.log(`App: showLinkPrompt(${link})`);
      this.shareLink = link;
      this.shareLinkPrompt = true;
    },
    async addFollowing() {
      window.ipc.send("follow", this.idToFollow);
      this.idToFollow = "";
    },
    async removeFollowing() {
      window.ipc.send("unfollow", this.idToUnollow);
      this.idToUnollow = "";
    },
  },
});
</script>
<style scoped>
.root-container {
  padding-left: 16.5%;
  padding-right: 16.5%;
  position: relative;
}
</style>
