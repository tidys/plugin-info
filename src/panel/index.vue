<template>
  <div class="panel">
    <div class="left">
      <div>Project Plugin[{{ treeData.length }}]:</div>
      <CCTree style="min-width: 100px; flex: 1" :value="treeData" @node-click="onNodeClick"></CCTree>
    </div>
    <div class="right">
      <div class="content" v-if="menus.length">
        菜单信息：
        <div>
          <div v-for="(menu, index) in menus" :key="index">
            <div class="path">{{ menu.path }}</div>
          </div>
        </div>
        <div style="flex: 1"></div>
        <div style="display: flex; flex-direction: row; justify-content: flex-end">
          <CCButton @confirm="onClickBtn" color="red">删除插件</CCButton>
        </div>
      </div>
    </div>
    <CCDialog></CCDialog>
  </div>
</template>
<script lang="ts">
import { defineComponent, onMounted, ref, provide, nextTick } from "vue";
import PluginConfig from "../../cc-plugin.config";
import ccui from "@xuyanfeng/cc-ui";
import CCP from "cc-plugin/src/ccp/entry-render";
import { existsSync, readdirSync, readFileSync, statSync } from "fs";
import { join } from "path";
import { ITreeData } from "@xuyanfeng/cc-ui/types/cc-tree/const";
const { CCInput, CCButton, CCTree, CCDialog } = ccui.components;
import { CocosPluginV2, CocosPluginV3 } from "cc-plugin/src/declare";
import { removeSync } from "fs-extra";

export interface PkgMenu {
  /**
   * 菜单的路径
   */
  path: string;
}
class PkgInfo {
  /**
   * 插件的目录
   */
  dir: string;
  /**
   * 插件名字
   */
  name: string;
  menus: Array<PkgMenu> = [];
}

export default defineComponent({
  name: "index",
  components: { CCButton, CCTree, CCDialog },
  setup(props, { emit }) {
    onMounted(() => {
      updatePkg();
    });
    const pkgArray: PkgInfo[] = [];
    function transformMenu(arr: string[]) {
      const fullPath = [];
      arr.forEach((aa) => {
        aa.split("/").forEach((item) => {
          const flag = "i18n:";
          if (item.startsWith(flag)) {
            const key = item.slice(flag.length);
            //@ts-ignore
            const ret = CCP.Adaptation.Env.isPluginV2 ? Editor.i18n.t(key) : Editor.I18n.t(key);
            fullPath.push(ret);
          } else {
            fullPath.push(item);
          }
        });
      });
      return fullPath.join("/");
    }

    function updatePkg() {
      pkgArray.length = 0;
      const dir = join(CCP.Adaptation.Project.path, CCP.Adaptation.Env.pluginDirectory);
      const files = readdirSync(dir);
      for (let i = 0; i < files.length; i++) {
        const file = files[i];
        if (!statSync(join(dir, file)).isDirectory()) {
          continue;
        }
        const pkgJson = join(dir, file, "package.json");
        if (!existsSync(pkgJson)) {
          continue;
        }
        const pkgInfo = new PkgInfo();
        pkgInfo.dir = join(dir, file);
        const data = JSON.parse(readFileSync(pkgJson, "utf-8"));
        if (CCP.Adaptation.Env.isPluginV2) {
          const pkg: CocosPluginV2 = data as CocosPluginV2;
          pkgInfo.name = pkg.name;
          const pkgMenu = pkg["main-menu"];
          if (pkgMenu) {
            for (const key in pkgMenu) {
              pkgInfo.menus.push({ path: transformMenu([key]) });
            }
          }
        } else if (CCP.Adaptation.Env.isPluginV3) {
          const pkg: CocosPluginV3 = data as CocosPluginV3;
          pkgInfo.name = pkg.name;
          if (pkg.contributions && pkg.contributions.menu) {
            pkg.contributions.menu.forEach((menu) => {
              const ret = transformMenu([menu.path, menu.label]);
              pkgInfo.menus.push({ path: ret });
            });
          }
        }
        pkgArray.push(pkgInfo);
      }
      pkgArray.forEach((pkg) => {
        treeData.value.push({
          text: pkg.name,
          userData: pkg,
        });
      });
    }
    const treeData = ref<ITreeData[]>([]);
    const msg = ref(PluginConfig.manifest.name);
    const menus = ref<PkgMenu[]>([]);
    let curPkg: PkgInfo | null = null;
    return {
      treeData,
      msg,
      menus,
      onClickBtn() {
        if (curPkg) {
          // 确认在位置1
          const ret = CCP.Adaptation.Dialog.message({ message: `删除${curPkg.name}插件`, buttons: ["取消", "确认"] });
          debugger;
          if (!!ret) {
            removeSync(curPkg.dir);
            updatePkg();
            menus.value.length = 0;
          }
        }
      },
      onNodeClick(data: ITreeData) {
        const pkgInfo = data.userData as PkgInfo;
        curPkg = pkgInfo;
        menus.value.length = 0;
        pkgInfo.menus.forEach((menu) => {
          menus.value.push({ path: menu.path });
        });
      },
    };
  },
});
</script>

<style scoped lang="less">
.panel {
  display: flex;
  flex-direction: row;
  width: 100%;
  height: 100%;
  overflow: hidden;
  .left {
    display: flex;
    flex-direction: column;
    flex: 1;
  }
  .right {
    flex: 2;
    padding-left: 10px;
    display: flex;
    flex-direction: column;
    .content {
      display: flex;
      flex: 1;
      flex-direction: column;
      .label {
        background-color: #7b7b7b;
        padding: 3px;
      }
      .path {
        background-color: #444;
        padding: 3px;
      }
    }
  }
}
</style>
