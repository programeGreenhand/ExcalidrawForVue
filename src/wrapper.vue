<template>
  <div class="container">
    
    <div class="excalidraw" id="excalidraw">

    </div>

    
  </div>
</template>

<script>
import { createRoot } from "react-dom/client";
import React from "react";
import { Excalidraw} from "@excalidraw/excalidraw";
import "@excalidraw/excalidraw/index.css";

let root = null;
let excalidrawAPI = null;

export default {
  data() {
    return {
      appState: {
        viewBackgroundColor: '#fff',
        currentItemStrokeColor: '#000000',
        currentItemBackgroundColor: '#ffffff',
        activeTool: 'selection',
        theme: 'light',
        zoom: 1,
        
      },
      
      
    };
  },
  mounted() {
    this.initializeExcalidraw();
  },
  unmounted() {
    if (root) {
      root.unmount();
    }
  },
  methods: {
    initializeExcalidraw() {
      // 从本地存储恢复数据
      const elements = JSON.parse(localStorage.getItem("excalidraw-elements") || "[]");
      const libs = JSON.parse(localStorage.getItem("excalidraw-libs") || "[]");
      const savedState = JSON.parse(localStorage.getItem("excalidraw-state") || "{}");

      
      
      // 合并保存的状态和默认状态
      const initialState = { ...this.appState, ...savedState };
      
      root = createRoot(document.getElementById("excalidraw"));
      root.render(
        React.createElement(Excalidraw, {
          name: "我的画板",
          initialData: {
            elements: elements,
            libraryItems: libs,
            appState: initialState
          },
          langCode: "zh-CN",

          UIOptions: {
            canvasActions: {
              loadScene: false,       // 场景加载按钮
              saveScene: false,       // 场景保存按钮
              export: false,          // 导出按钮（PNG/SVG）
              toggleTheme: false,     // 主题切换按钮
              clearCanvas: false,     // 清空画布
              saveAsImage: false,     // 另存为图片
              saveToActiveFile: false // 保存到当前文件
            },
            tools: {
              image: true,           // 图片工具
              frame: false,           // 画框工具
              laser: false            // 激光笔工具
            }
          },
          
          
          onChange: this.onChange,
          onLibraryChange: this.onLibraryChange,
          excalidrawAPI: (api) => {
            excalidrawAPI = api;
            window.excalidrawAPI = api; // 可选: 全局访问 API
          }
        })
      );
    },
    
    save() {
      if (excalidrawAPI) {
        const appState = excalidrawAPI.getAppState();
        const elements = excalidrawAPI.getSceneElements();
        
        // 移除不需要保存的属性
        const { collaborators, ...stateToSave } = appState;
        
        localStorage.setItem("excalidraw-state", JSON.stringify(stateToSave));
        localStorage.setItem("excalidraw-elements", JSON.stringify(elements));
        
        
      }
    },

    
    onChange(elements) {
      localStorage.setItem("excalidraw-elements", JSON.stringify(elements));
    },
    
    onLibraryChange(libraryItems) {
      localStorage.setItem("excalidraw-libs", JSON.stringify(libraryItems));
    }
  }
};
</script>

<style scoped lang="scss">
#dropdown-menu-button{
  display: none;
}

.container {
  width: 100%;
  height: 100vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  
  .header {
    height: 3rem;
    line-height: 3rem;
    padding: 0 1rem;
    font-size: 1.2rem;
    background-color: #038fe5;
    color: white;
    display: flex;
    justify-content: space-between;
    align-items: center;
    
    button {
      padding: 0.3rem 0.8rem;
      background-color: white;
      color: #038fe5;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-weight: bold;
      
      &:hover {
        background-color: #f0f0f0;
      }
    }
  }
  
  .footer {
    height: 2rem;
    line-height: 2rem;
    text-align: center;
    background-color: #038fe5;
    color: white;
  }
  
  .excalidraw {
    flex-grow: 1;
    height: 100%;
  }
}

/* 修复可能的 Excalidraw UI 样式冲突 */
:global(.excalidraw) {
  --sat: env(safe-area-inset-top);
  --sab: env(safe-area-inset-bottom);
  --sal: env(safe-area-inset-left);
  --sar: env(safe-area-inset-right);
  z-index: 0;
}

:global(.ToolIcon_size_medium) {
  --button-size: 1.8rem !important;  /* 按钮容器大小 */
  --icon-size: 1.2rem !important;    /* 实际图标尺寸 */
}

/* 在 <style> 中添加 */
// 隐藏指定工具按钮
// :global(.excalidraw [id="V4wZu3j85ZyGMmPW9n2KS-undefined"]) { /* 锁定工具 */
//   display: none !important;
// }
// .excalidraw .sidebar-trigger__label-element{
//   display: none;
// }
</style>