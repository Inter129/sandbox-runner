<script lang="ts">
  import * as monaco from "monaco-editor";
  import editorWorker from "monaco-editor/esm/vs/editor/editor.worker?worker";
  import tsWorker from "monaco-editor/esm/vs/language/typescript/ts.worker?worker";
  import { onDestroy, onMount } from "svelte";
  import { Pane, Splitpanes } from "svelte-splitpanes";
  import * as prettier from "prettier";
  import babel from "prettier/plugins/babel";
  import esTree from "prettier/plugins/estree";

  import "./app.css";
  import theme from "./assets/theme";

  let editorElement: HTMLDivElement;
  let editor: monaco.editor.IStandaloneCodeEditor;
  let model: monaco.editor.ITextModel;

  // date, message, color
  let logs: [string, string, string][] = [];

  function loadCode(code: string, language: string) {
    model = monaco.editor.createModel(code, language);

    editor.setModel(model);
  }
  const log = (message: string, color = "#fff") => {
    const date = new Date();
    // format as HH:MM:SS
    const time = `${date.getHours().toString().padStart(2, "0")}:${date.getMinutes().toString().padStart(2, "0")}:${date.getSeconds().toString().padStart(2, "0")}`;
    logs = [...logs, [time, message, color]];

    // scroll to bottom
    const terminal = document.querySelector(".terminal") as HTMLDivElement;
    terminal.scrollTop = terminal.scrollHeight;
  };

  onMount(async () => {
    self.MonacoEnvironment = {
      getWorker: function (_: any, label: string) {
        if (label === "typescript" || label === "javascript") {
          return new tsWorker();
        }
        return new editorWorker();
      },
    };

    monaco.languages.typescript.typescriptDefaults.setEagerModelSync(true);
    monaco.editor.defineTheme("theme", theme as any);
    monaco.editor.setTheme("theme");

    editor = monaco.editor.create(editorElement, {
      automaticLayout: true,
      theme: "theme",
    });

    loadCode(
      `// 여기에 코드를 작성할 수 있어요!
console.log("Hello, World!");
console.info("Hello, World!");
console.error("Hello, World!");
console.warn("Hello, World!");`,
      "javascript"
    );
  });

  onDestroy(() => {
    monaco?.editor.getModels().forEach((model) => model.dispose());
    editor?.dispose();
  });

  const save = async () => {
    // save to local storage
    log("로그 | 코드를 저장하는 중입니다...");
    const code = model.getValue();
    const formattedCode = await prettier.format(code, {
      parser: "babel",
      plugins: [babel, esTree],
    });
    localStorage.setItem("code", formattedCode);
    loadCode(formattedCode, "javascript");
    log("성공 | 코드가 성공적으로 저장되었습니다.", "#C3E88D");
  };

  const clearConsole = () => {
    logs = [];
  };

  const load = () => {
    // load from local storage
    log("로그 | 코드를 불러오는 중입니다...");
    const code = localStorage.getItem("code") || "";
    loadCode(code, "javascript");
    log("성공 | 코드를 성공적으로 불러왔습니다.", "#C3E88D");
  };

  const download = () => {
    // download as file
    log("로그 | 코드를 다운로드하는 중입니다...");
    const code = model.getValue();
    const blob = new Blob([code], { type: "text/plain" });
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = "code.js";
    a.click();
    URL.revokeObjectURL(url);
    log("성공 | 코드가 성공적으로 다운로드되었습니다.", "#C3E88D");
  };

  const loadFromFile = () => {
    // load from file
    const input = document.createElement("input");
    input.type = "file";
    input.accept = ".js";
    log("로그 | 파일을 선택해주세요.");
    input.onchange = (e) => {
      const file = (e.target as HTMLInputElement).files?.[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          const code = e.target?.result as string;
          loadCode(code, "javascript");
          log("성공 | 파일을 성공적으로 불러왔습니다.", "#C3E88D");
        };
        reader.readAsText(file);
      }
    };
    input.click();
  };

  const run = () => {
    clearConsole();
    log(
      "로그 | =================== 코드 실행중 ===================",
      "#727779"
    );
    let code2run = `
    const __send__ = (type, data) => {
      parent.postMessage({ type, data }, "*");
    };
    const __cslmap__ = (dt) => {
      try {
        return ["boolean", "number", "string", "bigint"].includes(typeof dt) ? dt.toString() : JSON.stringify(dt);
      } catch (e) {
        return dt.toString();
      }
    }
    console = {
      log: (...args) => __send__("log", args.map(__cslmap__).join(" ")),
      error: (...args) => __send__("error", args.map(__cslmap__).join(" ")),
      warn: (...args) => __send__("warn", args.map(__cslmap__).join(" ")),
      info: (...args) => __send__("info", args.map(__cslmap__).join(" ")),
    }
    async function __run__() {
      try {
        ${model.getValue()}
        __send__("done", {s: true});
      } catch (e) {
        __send__("done", {s: false, e: e.toString()});
      }
    }
    __run__();
    `;

    let iframe = document.createElement("iframe");
    iframe.style.display = "none";
    document.body.appendChild(iframe);

    // let iframeHTML: string = [
    //   "<html>",
    //   "<head>",
    //   `<script src="https://cdnjs.cloudflare.com/ajax/libs/fetch-jsonp/1.0.6/fetch-jsonp.min.js"></scr` +
    //     `ipt>`,
    //   "<script>",
    //   code2run,
    //   "</sc" + "ript>", // ㅁㅊ?
    //   "</head>",
    //   "<body>",
    //   "</body>",
    //   "</html>",
    // ].join("");

    iframe.contentWindow.document.open();
    iframe.contentWindow.document.appendChild(
      iframe.contentWindow.document.createElement("script")
    ).textContent = code2run;
    iframe.contentWindow.document.close();

    const messageListener = (e: MessageEvent) => {
      if (e.source === iframe.contentWindow) {
        const { type, data } = e.data;
        if (typeof type != "string") return;
        if (typeof data == "undefined") return;
        if (type === "done") {
          log(
            "로그 | =================== 코드 실행끝 ===================",
            "#727779"
          );
          log(
            data.s
              ? "성공 | 코드가 정상적으로 실행되었습니다."
              : "오류 | 코드 실행 중 오류가 발생했습니다.",
            data.s ? "#C3E88D" : "#FF6464"
          );
          if (!data.s) {
            log(data.e, "#FF6464");
          }
          document.body.removeChild(iframe);
          window.removeEventListener("message", messageListener);
        }
      }
    };
    window.addEventListener("message", messageListener);
  };

  onMount(() => {
    const messageListener = (e: MessageEvent) => {
      const { type, data } = e.data;
      if (typeof type != "string") return;
      if (typeof data == "undefined") return;
      if (type == "log") {
        log("로그 | " + data);
      }
      if (type == "error") {
        log("오류 | " + data, "#FF6464");
      }
      if (type == "warn") {
        log("경고 | " + data, "#FFCB6B");
      }
      if (type == "info") {
        log("정보 | " + data, "#81A7FB");
      }
    };
    window.addEventListener("message", messageListener);

    return () => window.removeEventListener("message", messageListener);
  });

  onMount(() => {
    const isMac = navigator.platform.toUpperCase().indexOf("MAC") >= 0;
    // ctrl s to save
    // ctrl r to run
    const saveHandler = (e: KeyboardEvent) => {
      if (e.key === "s" && (isMac ? e.metaKey : e.ctrlKey)) {
        e.preventDefault();
      }
      if (!e.repeat && e.key === "s" && (isMac ? e.metaKey : e.ctrlKey)) {
        save();
      }

      if (e.key === "r" && (isMac ? e.metaKey : e.ctrlKey)) {
        e.preventDefault();
      }
      if (!e.repeat && e.key === "r" && (isMac ? e.metaKey : e.ctrlKey)) {
        run();
      }
    };
    window.addEventListener("keydown", saveHandler);

    log(
      "정보 | 콘솔에 오신 것을 환영합니다! 이곳에서는 코드의 실행 결과를 확인할 수 있습니다."
    );
    log("정보 | Ctrl + S를 눌러 코드를 이쁘게 하고 저장 할 수 있습니다.");
    log("정보 | Ctrl + R를 눌러 코드를 실행 할 수 있습니다.");

    return () => window.removeEventListener("keydown", saveHandler);
  });
</script>

<Splitpanes style="height: 100dvh">
  <Pane minSize={20}>
    <div
      style="width: 100%; height: 100dvh; display: flex; flex-direction: column; background-color: #242A2E;"
    >
      <div style="flex-grow: 1;" bind:this={editorElement}></div>
    </div>
  </Pane>
  <Pane>
    <div
      style="width: 100%; height: 100dvh; display: flex; flex-direction: column; background-color: #242A2E; padding: .75rem; gap: .75rem; box-sizing: border-box;"
    >
      <div>
        <button on:click={save}>저장</button>
        <button on:click={load}>불러오기</button>
        <button on:click={download}>다운로드</button>
        <button on:click={loadFromFile}>파일로부터 불러오기</button>
        <button on:click={run}>실행 (Ctrl + R)</button>
      </div>

      <!-- Terminal -->
      <div
        style="background: #1A2227; color: #fff; padding: 1rem; width: 100%; overflow-y: auto; flex-grow: 1; box-sizing: border-box; border-radius: .5rem"
        class="terminal"
      >
        {#each logs as [date, message], i}
          <div
            style={`
          font-family: 'Fira Code', monospace;
          font-size: .75rem;
          line-height: 1.5;
          white-space: pre-wrap;
          word-break: break-all;
          color: ${logs[i][2] || "#fff"};
        `}
          >
            {date} | {message}
          </div>
        {/each}
      </div>
    </div>
  </Pane>
</Splitpanes>
