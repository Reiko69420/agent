<script>
  import { onDestroy, onMount } from "svelte";
  import ControlPanel from "$lib/components/ControlPanel.svelte";
  import MessageContainer from "$lib/components/MessageContainer.svelte";
  import MessageInput from "$lib/components/MessageInput.svelte";
  import BrowserWidget from "$lib/components/BrowserWidget.svelte";
  import TerminalWidget from "$lib/components/TerminalWidget.svelte";
  import * as Resizable from "$lib/components/ui/resizable/index.js";
  import { toast } from "svelte-sonner";

  import {
    fetchInitialData,
    fetchAgentState,
    checkInternetStatus,
    socket,
  } from "$lib/api";
  import { messages, tokenUsage, agentState } from "$lib/store";
    import EditorWidget from "../lib/components/EditorWidget.svelte";

  let resizeEnabled =
    localStorage.getItem("resize") &&
    localStorage.getItem("resize") === "enable";
  let prevMonologue = null;
  
  onMount(() => {
    // localStorage.clear();
    const load = async () => {
      await fetchInitialData();

      await fetchAgentState();
      // await fetchMessages();
      await checkInternetStatus();
    };
    load();

    prevMonologue = $agentState?.internal_monologue;

    socket.emit("socket_connect", { data: "frontend connected!" });
    socket.on("socket_response", function (msg) {
      console.log(msg);
    });

    socket.on("server-message", function (data) {
      console.log("server-message: ", data);
      messages.update((msgs) => [...msgs, data["messages"]]);
    });

    socket.on("agent-state", function (state) {
      const lastState = state[state.length - 1];
      agentState.set(lastState);
      console.log("server-state: ", lastState);
    });

    socket.on("tokens", function (tokens) {
      tokenUsage.set(tokens["token_usage"]);
    });

    agentState.subscribe((state) => {
      function handleMonologueChange(newValue) {
        if (newValue) {
          if(newValue == "Writing code..." || newValue == "Agent has completed the task."){
            toast.success(newValue);
          }
          else if(newValue.includes("error")) {
            toast.error(newValue);
          } 
          else {
            toast.message(newValue);
          }
        }
      }
      if (state && state.internal_monologue && state.internal_monologue !== prevMonologue) {
        handleMonologueChange(state.internal_monologue);
        prevMonologue = state.internal_monologue;
      }
    });
  });

  onDestroy(() => {
    if (socket.connected) {
      socket.off("socket_response");
      socket.off("server-message");
      socket.off("agent-state");
      socket.off("tokens");
    }
  });
</script>

<div class="flex h-full flex-col flex-1 gap-4 p-4 overflow-hidden">
  <ControlPanel />

  <div class="flex flex-1 max-w-[95vw] overflow-x-auto">
    <Resizable.PaneGroup direction="horizontal" class={"min-w-[140vw] snap-mandatory snap-x"} autoSaveId="default">
      <Resizable.Pane defaultSize={40} class="snap-center">
        <div class="flex flex-col gap-2 w-full h-full pr-4">
          <MessageContainer />
          <MessageInput />
        </div>
      </Resizable.Pane>
      {#if resizeEnabled}
        <Resizable.Handle />
      {/if}
      <Resizable.Pane defaultSize={40} class="snap-center">
        <Resizable.PaneGroup direction="vertical">
          <Resizable.Pane defaultSize={50}>
            <div class="flex h-full items-center justify-center p-2">
              <BrowserWidget />
            </div>
          </Resizable.Pane>
          {#if resizeEnabled}
            <Resizable.Handle />
          {/if}
          <Resizable.Pane defaultSize={50}>
            <div class="flex h-full items-center justify-center p-2">
              <TerminalWidget />
            </div>
          </Resizable.Pane>
        </Resizable.PaneGroup>
      </Resizable.Pane>
      {#if resizeEnabled }
        <Resizable.Handle />
      {/if}
      <Resizable.Pane defaultSize={50} class="snap-center">
        <div class="flex flex-col gap-2 w-full h-full pr-4 p-2">
          <EditorWidget />
        </div>
      </Resizable.Pane>
    </Resizable.PaneGroup>
    </div>
</div>