<template>
  <div class="flex-1 flex flex-col md:flex-row pt-2">
    <div class="flex flex-col flex-1">
      <tinker-editor class="flex-1" v-model="code" language="php-x" theme="one-light"></tinker-editor>
      <div class="py-2 px-3 flex justify-center md:justify-start">
        <div class="flex flex-row items-center">
          <input type="checkbox" class="input-checkbox" id="autoTinker" :checked="autoTinker" @change="enableautoTinker($event.target.checked)" />
          <label class="ml-2 text-sm text-gray-600 dark:text-white" for="autoTinker">Auto Tinker</label>
        </div>
        <kit-button @click.native="executeTinker" class="ml-auto">Tinker</kit-button>
      </div>
    </div>
    <tinker-editor class="flex-1" v-model="output" language="php-x" theme="one-light" :options="outputOptions"></tinker-editor>
  </div>
</template>

<script>
import TinkerEditor from "@/components/TinkerEditor.vue";
import KitButton from "@/components/KitButton.vue";
import { mapState } from "vuex";
import { spawn } from "child_process";
import initTinker from "@/lib/tinker.js";
initTinker();

export default {
  name: "Tinker",
  components: { TinkerEditor, KitButton },
  data() {
    return {
      outputOptions: {
        readOnly: true,
        wordWrap: "wordWrapColumn",
        wordWrapColumn: 100
      },
      timeOut: 0
    };
  },
  computed: {
    ...mapState(["dir"]),
    theme() {
      return this.$store.state.dark ? "dracula" : "one-light";
    },
    autoTinker: {
      set(value) {
        this.$store.state.autoTinker = value;
      },
      get() {
        return this.$store.state.autoTinker;
      }
    },
    code: {
      set(value) {
        this.$store.state.code = value;
      },
      get() {
        return this.$store.state.code;
      }
    },
    output: {
      set(value) {
        this.$store.state.output = value;
      },
      get() {
        return this.$store.state.output;
      }
    }
  },
  watch: {
    code: function () {
      if (!this.$store.state.autoTinker) {
        return;
      }
      clearTimeout(this.timeOut);
      this.timeOut = setTimeout(() => {
        this.executeTinker();
      }, 800);
    }
  },
  methods: {
    enableautoTinker(value) {
      this.$store.state.autoTinker = typeof this.$store.state.autoTinker === "undefined" ? false : value;
    },
    executeTinker() {
      if (this.$store.state.php === "") {
        return window.Electron.dialogPhpNotFound();
      }

      this.$store.state.output = "";
      this.$store.state.tinkering = true;

      const tinker = spawn(this.$store.state.php, ["artisan", "tinker"], { cwd: this.dir });
      tinker.stdout.setEncoding("utf-8");
      tinker.stdin.write(this.code);
      tinker.stdin.end();

      tinker.stdout.on("data", (data) => {
        this.$store.state.output += data;
        tinker.kill();
      });
      tinker.stderr.on("data", (err) => {
        this.$store.state.output = err.toString().replace(/<[^>]*>?/gm, "");
        tinker.kill();
      });
      tinker.on("error", (err) => {
        this.$store.state.output = err.toString();
      });
      tinker.on("close", () => {
        this.$store.state.tinkering = false;
      });
    }
  }
};
</script>

<style></style>
