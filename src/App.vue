<template>
  <div class="container-wrap" v-bind:class="{ lspdark: opts.isDark }" @click="_onClickOutside">
    <div class="container-inner shadow-lg" v-if="ready" :style="{ left: left + 'px', top: top + 'px' }">
      <div class="_opener" v-if="currentPage" @click="_onClickOpenCurrentPage">
        Edit current page
      </div>
      <div class="_opener" @click="_onClickOpenConfig">Edit config.edn</div>
      <div class="_opener" @click="_onClickOpenCSS">Edit custom.css</div>
      <div class="_opener" @click="_onClickOpenCurrentGraph">
        Open graph folder
      </div>
    </div>
  </div>
</template>

<script>
import { setBlockTracking } from 'vue';

function generateUrl(path, line_num = 0) {
  const { distro } = logseq.settings;
  const protocol = distro === "stable" ? "vscode" : distro === "insiders" ? "vscode-insiders" : "vscodium";
  console.log('in url', line_num);
  return `${protocol}://file/` + encodeURIComponent(path) + `:${line_num}:0` + "?windowId=_blank";
}

async function openConfig(name) {
  const graph = await logseq.App.getCurrentGraph();
  window.open(
    generateUrl(graph.url.replace("logseq_local_", "") + "/logseq/" + name)
  );
}

async function openGraph() {
  const graph = await logseq.App.getCurrentGraph();
  window.open(generateUrl(graph.url.replace("logseq_local_", "")));
}

async function getAncestorPageOfCurrentBlock() {
  const block = await logseq.Editor.getCurrentBlock();
  return block?.page;
}

async function findFile(fileId) {
  const graph = await logseq.App.getCurrentGraph();

  const matches = await logseq.DB.datascriptQuery(
    `[:find ?file
                :where
                [?b :file/path ?file]
                [(== ?b ${fileId})]
            ]`
  );

  if (matches && matches.length > 0) {
    const file = graph.url.replace("logseq_local_", "") + "/" + matches[0][0];
    return file;
  } else {
    return null;
  }
}



async function openPageInVSCode(line_number = 0) {
  const currentPage = await logseq.Editor.getCurrentPage();
  if (currentPage && currentPage.file) {
    const fileId = currentPage.file.id;
    const file = await findFile(fileId);

    if (file) {
      window.open(generateUrl(file, line_number));
      return;
    }
  }

  const ansetor = await getAncestorPageOfCurrentBlock();
  if (ansetor) {
    const page = await logseq.Editor.getPage(ansetor.id);
    if (page && page.file) {
      const fileId = page.file.id;
      const file = await findFile(fileId);
      if (file) {
        window.open(generateUrl(file, line_number));
        return;
      }
    }
  }
}

function reorder_children(children) {
  let id_chains = children.map(child => child.left);

  children.sort((a, b) => {
    let indexA = id_chains.indexOf(a.left);
    let indexB = id_chains.indexOf(b.left);
    if (indexA < indexB) {
      return -1;
    } else if (indexA > indexB) {
      return 1;
    } else {
      return 0;
    }
  });
}

function count_line_in_block(block, line, reorder = false) {
  let count = 0;
  let found_line = false;

  if (block.children.length > 0) {
    if (reorder) {
      reorder_children(block.children);
    }
    block.children.forEach(child => {
      let search_subblock = count_line_in_block(child, line);
      if (!found_line) {
        count += search_subblock.lineCount;
        found_line = found_line || search_subblock.hasLine;
      }
    });
    console.log(block, block.content.split('\n').length, "old count", count, found_line);
    return { lineCount: block.content.split('\n').length + count, hasLine: block.id === line.id || found_line };
  } else {
    console.log(block, block.content.split('\n').length);
    return { lineCount: block.content.split('\n').length, hasLine: block.id === line.id };
  }
}

// function check_has_line(block, line) {
//   if (block.children.length > 0) {
//     block.children.forEach(child => {
//       if (check_has_line(child, line)) {
//         return true;
//       }
//     });
//     return false;
//   } else {
//     return block.content.includes(line);
//   }

// }

async function openCurrentLine() {

  let count = 0;

  const currentPage = await logseq.Editor.getCurrentPage();
  let all_blocks = await logseq.Editor.getCurrentPageBlocksTree();
  console.log(all_blocks.length === 1 && !('content' in all_blocks[0]));
  console.log('content' in all_blocks[0]);
  if (all_blocks.length === 1 && !('content' in all_blocks[0])) {
    const ancestor = await getAncestorPageOfCurrentBlock();
    console.log(ancestor);
    const page = await logseq.Editor.getPage(ancestor.id);
    console.log(page);
    all_blocks = await logseq.Editor.getPageBlocksTree(page.name);
    console.log("retrieved all blocks", ancestor, page, all_blocks);
  }

  let curb = await logseq.Editor.getCurrentBlock();
  console.log("curb", curb);
  console.log("all_blocks", all_blocks);

  // let subcount = 
  for (let index = 0; index < all_blocks.length; index++) {
    const block = all_blocks[index];
    let subcount = count_line_in_block(block, curb);
    // if not found line, then add line count
    if (subcount.hasLine) {
      subcount = count_line_in_block(block, curb, true);
      count += subcount.lineCount;
      break;
    } else {
      count += subcount.lineCount;
    }
  }

  await openPageInVSCode(count);
}

async function registerShortcuts() {
  logseq.App.registerCommandPalette({
    key: `Open_current_line_in_default_editor`,
    label: "Open current line in default editor",
    keybinding: {
      binding: logseq.settings.key_open_line,
      mode: "global",
    }
  },
    openCurrentLine
  );
  logseq.App.registerCommandPalette({
    key: `Open_current_page_in_default_editor`,
    label: "Open current page in default editor",
    keybinding: {
      binding: logseq.settings.key_open_page,
      mode: "global",
    }
  },
    () => openPageInVSCode(0)
  );

  logseq.App.registerCommandPalette({
    key: `Open_graph_folder_in_default_editor`,
    label: "Open graph folder in default editor",
    keybinding: {
      binding: logseq.settings.key_open_graph,
      mode: "global",
    }
  },
    openGraph
  );
}

export default {
  name: "App",

  data() {
    return {
      ready: false,
      left: 0,
      top: 0,
      currentPage: false,
      opts: {
        isDark: false,
      },
    };
  },

  mounted() {
    logseq.App.getUserConfigs().then(
      (c) => (this.opts.isDark = c.preferredThemeMode === "dark")
    );

    logseq.App.onThemeModeChanged(({ mode }) => {
      this.opts.isDark = mode === "dark";
    });

    logseq.once("ui:visible:changed", ({ visible }) => {
      visible && (this.ready = true);
    });

    const checkCurrentPage = async () => {
      const currentPage = await logseq.Editor.getCurrentPage();

      if (currentPage && currentPage.file) {
        this.currentPage = true;
      } else {
        const ansestor = await getAncestorPageOfCurrentBlock();
        if (ansestor) {
          this.currentPage = true;
        } else {
          this.currentPage = false;
        }
      }
    };

    logseq.on("ui:visible:changed", ({ visible }) => {
      if (visible) {
        const el = top.document.querySelector(`a#open-in-code-anchor`);
        const rect = el.getBoundingClientRect();
        this.left = rect.left - 50;
        this.top = rect.top + 30;
        checkCurrentPage();
      }
    });

    logseq.App.onRouteChanged(({ path }) => {
      checkCurrentPage();
    });

    checkCurrentPage();

    registerShortcuts();
  },

  methods: {
    _onClickOutside({ target }) {
      const inner = target.closest(".container-inner");

      !inner && logseq.hideMainUI();
    },
    _onClickOpenCurrentPage() {
      openPageInVSCode();
    },
    _onClickOpenCurrentGraph() {
      openGraph();
    },
    _onClickOpenConfig() {
      openConfig("config.edn");
    },
    _onClickOpenCSS() {
      openConfig("custom.css");
    },
  },
};
</script>
