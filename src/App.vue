<template>
  <div class="container">
    <!-- 画布区域 -->
    <div 
      class="excalidraw" 
      id="excalidraw"
      @dragover.prevent 
      @drop="handleDropOnCanvas"
    ></div>

    <!-- 控制面板 -->
    <div class="control-panel">
      <!-- 线条粗细控制 -->
      <div class="slider-container">
        <span>线条粗细</span>
        <el-slider 
          v-model="strokeWidth" 
          :min="0.3" 
          :max="15" 
          :step="1"
          @change="updateStrokeWidth"
        />
        <span>{{ strokeWidth }}px</span>
      </div>
      
      <!-- 图库按钮 -->
      <el-button 
        type="primary" 
        icon="el-icon-picture" 
        @click="()=> drawer = !drawer"
      >
        图库 {{ drawer ? '>>' : '<<' }}
      </el-button>

      <!-- 其他功能按钮 -->
      <div class="action-buttons">
        <button @click="save">保存画布</button>
        <button @click="exportAsJson">导出为JSON</button>
        <input 
          type="file" 
          id="jsonImport" 
          accept=".json" 
          @change="importFromJson"
          style="display: none"
        />
        <button @click="triggerImport">导入JSON</button>
        <button @click="toggleCustomUI">
          {{ showCustomUI ? '简化界面' : '完整界面' }}
        </button>
      </div>
    </div>
    
    <!-- 图库抽屉 -->
   
      <div v-show="drawer" class="gallery-content">
        <!-- <div class="drag-instructions">
          <span style="color: #038fe5;">拖拽素材添加到画布</span>
        </div> -->
        
        <div class="gallery-grid">
          <div 
            v-for="(image, index) in galleryImages" 
            :key="image.id" 
            class="gallery-item"
          >
            <div class="image-container">
              <img :src="base64Images[image.id]" :alt="image.name" />
            </div>
            
          </div>
        </div>
      </div>
 
  </div>
</template>

<script>
import { createRoot } from "react-dom/client";
import React from "react";
import { Excalidraw } from "@excalidraw/excalidraw";
import "@excalidraw/excalidraw/index.css";



let root = null;
let excalidrawAPI = null;

export default {
  data() {
    return {
      showCustomUI: true,
      drawer: false,
      strokeWidth: 1,
      isLoading: false,
      showDropSuccess: false,
      successMessage: "",

      //此处为写定的图像来源url 为什么可以实现拖拽到画布，貌似是convas有组织不同来源图片进行绘制的条件

      //要都转为base64格式 然后放入src才行！！！！！！

      //重点！！！！base64格式
      galleryImages: [
        { id: 1, name: "山脉", url: "https://images.unsplash.com/photo-1464822759023-fed622ff2c3b?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=600&q=80" },
        { id: 2, name: "海滩", url: "https://images.unsplash.com/photo-1505228395891-9a51e7e86bf6?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=600&q=80" },
        { id: 3, name: "森林", url: "https://images.unsplash.com/photo-1448375240586-882707db888b?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=600&q=80" },
        { id: 4, name: "沙漠", url: "https://images.unsplash.com/photo-1418065460487-3e41a6c84dc5?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=600&q=80" },
        { id: 5, name: "城市", url: "https://images.unsplash.com/photo-1477959858617-67f85cf4f1df?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=600&q=80" },
        { id: 6, name: "星空", url: "https://images.unsplash.com/photo-1419242902214-272b3f66ee7a?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=600&q=80" },
      ],
      base64Images: {},
      appState: {
        viewBackgroundColor: '#fff',
        currentItemStrokeColor: '#000000',
        currentItemBackgroundColor: '#ffffff',
        currentItemStrokeWidth: 1,
        activeTool: 'selection',
        theme: 'light',
        zoom: 1,
        uiOptions: {
          canvasActions: {
            export: false,
            loadScene: true,
            saveToActiveFile: true,
            saveAsImage: true,
            changeViewBackgroundColor: false,
            clearCanvas: true,
            help: false
          },
          tools: {
            image: true,
            eraser: false,
            arrow: false,
            text: false,
            line: false,
            rectangle: false,
            ellipse: false,
            selection: false,
            freedraw: false
          }
        }
      }
    };
  },
  async mounted() {
    await this.loadImages();
    this.initializeExcalidraw();
    this.applyCustomCSS();
  },
  beforeDestroy() {
    if (root) {
      root.unmount();
    }
  },
  methods: {


    //加载数据url，转为base64格式
    async loadImages() {
          this.isLoading = true;
          const imagesMap = {};
          
          try {
            // 严格使用 for...of 循环，与 React 一致
            for (const image of this.galleryImages) {
              // 完全相同的 fetch 逻辑
              const response = await fetch(image.url);
              // 相同的 blob 处理
              const blob = await response.blob();
              // 相同的 base64 转换
              const base64 = await this.convertBlobToBase64(blob);
              // 相同的存储方式（使用 id 作为键）
              imagesMap[image.id] = base64;
            }
            
            // 直接赋值，相当于 React 的 setBase64Images
            this.base64Images = imagesMap;
          } catch (err) {
            // 相同的错误处理
            console.error("图片加载失败:", err);
            // 添加用户反馈（Vue 方式）
          
          } finally {
            // 相同的加载状态重置
            this.isLoading = false;
            // 相同的日志输出
            console.log("所有图片加载完成");
          }
        },
  
  //将u图片转为base64格式
  // 完全相同的 convertBlobToBase64 方法
  convertBlobToBase64(blob) {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onloadend = () => resolve(reader.result);
      reader.onerror = reject;
      reader.readAsDataURL(blob);
    });
  },
    
  //初始化画布数据
    initializeExcalidraw() {
      // 从本地存储恢复数据
      const elements = JSON.parse(localStorage.getItem("excalidraw-elements") || "[]");
      const libs = JSON.parse(localStorage.getItem("excalidraw-libs") || "[]");
      const savedState = JSON.parse(localStorage.getItem("excalidraw-state") || "{}");
      
      // 如果本地存储有线宽值，使用它
      if (savedState.currentItemStrokeWidth) {
        this.strokeWidth = savedState.currentItemStrokeWidth;
      }
      
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
          onChange: this.onChange,
          onLibraryChange: this.onLibraryChange,
          UIOptions: this.appState.uiOptions,
          excalidrawAPI: (api) => {
            excalidrawAPI = api;
            window.excalidrawAPI = api;
          }
        })
      );
    },

    // 更新线条粗细
    updateStrokeWidth(value) {
      if (!excalidrawAPI) return;
      
      // 更新当前的线条粗细
      excalidrawAPI.updateScene({
        appState: {
          currentItemStrokeWidth: value
        }
      });
    },
    
    // 应用自定义CSS来隐藏更多UI元素
    applyCustomCSS() {
      // 创建一个样式元素
      const styleEl = document.createElement('style');
      styleEl.id = 'excalidraw-custom-styles';
      
      // 根据当前UI显示状态设置CSS
      this.updateCustomCSS(styleEl);
      
      // 添加到文档头部
      document.head.appendChild(styleEl);
    },
    
    // 更新自定义CSS
    updateCustomCSS(styleEl = document.getElementById('excalidraw-custom-styles')) {
      if (!styleEl) return;
      
      // 如果要显示完整UI，则清空样式
      if (this.showCustomUI) {
        styleEl.textContent = '';
        return;
      }
      
      // 否则，应用隐藏样式
      styleEl.textContent = `
        /* 隐藏库按钮 */
        
        /* 隐藏形状库面板 */
        .layer-ui__library {
          display: none !important;
        }
        
        /* 隐藏协作按钮 */
        .CollabButton {
          display: none !important;
        }
        
        /* 隐藏右上角菜单的一些项目 */
        .dropdown-menu-group-sessions,
        .dropdown-menu-item-export,
        .dropdown-menu-item-save,
        .dropdown-menu-item-load {
          display: none !important;
        }
        
        /* 隐藏缩放控制 */
        .zoom-actions {
          display: none !important;
        }
        
        .undo-button-container{
        display: none !important;
        }

        .redo-button-container{
        display: none !important;
        }
        
        .dropdown-menu-group {
        display: none !important;
        }

        .help-icon{
        display: none !important;
        }

        .default-sidebar-trigger{
        display: none !important;
        }

        button[data-testid="dropdown-menu-button"][type="button"]{
        display: none !important;
        }

        button[data-testid="search-menu-button"][type="button"]{
        display: none !important;
        }
        
        /* 隐藏右侧的属性面板中的一些高级选项 */
        .panel-row-horizontal > .ToolIcon[title="锁定"] {
          display: none !important;
        }
      `;
    },
    
    // 切换UI显示模式
    toggleCustomUI() {
      this.showCustomUI = !this.showCustomUI;
      this.updateCustomCSS();
    },
    
    //保存画布
    save() {
      if (excalidrawAPI) {
        const appState = excalidrawAPI.getAppState();
        const elements = excalidrawAPI.getSceneElements();
        
        // 移除不需要保存的属性
        const { collaborators, ...stateToSave } = appState;
        
        // 确保存储当前线宽
        stateToSave.currentItemStrokeWidth = this.strokeWidth;
        
        localStorage.setItem("excalidraw-state", JSON.stringify(stateToSave));
        localStorage.setItem("excalidraw-elements", JSON.stringify(elements));
        
        this.$message.success("保存成功!");
      }
    },
    
    //导出画布JSON数据
    exportAsJson() {
      if (excalidrawAPI) {
        const appState = excalidrawAPI.getAppState();
        const elements = excalidrawAPI.getSceneElements();
        
        // 移除不需要保存的属性
        const { collaborators, ...stateToSave } = appState;
        
        // 确保导出当前线宽
        stateToSave.currentItemStrokeWidth = this.strokeWidth;
        
        // 创建包含画布所有信息的对象
        const exportData = {
          elements: elements,
          appState: stateToSave,
          libraries: JSON.parse(localStorage.getItem("excalidraw-libs") || "[]"),
          files: excalidrawAPI.getFiles() || {}
        };
        
        // 转换为JSON并下载
        const dataStr = JSON.stringify(exportData, null, 2);
        //此处为画布Json数据的导出
        //在此提供向后端发送数据的接口
        /**如果需要发送到后端，可以在这里添加代码*/



        const dataUri = "data:application/json;charset=utf-8," + encodeURIComponent(dataStr);
        
        const exportFileName = `drawing-${new Date().toISOString().slice(0, 10)}.json`;
        
        const linkElement = document.createElement("a");
        linkElement.setAttribute("href", dataUri);
        linkElement.setAttribute("download", exportFileName);
        linkElement.click();
      }
    },
    
    //触发导入功能（Json画布数据导入)
    triggerImport() {
      document.getElementById("jsonImport").click();
    },
    
    importFromJson(event) {
      //此函数为处理从文件输入导入JSON数据
      ///以下实现为本地文件夹导入Json画布数据，
      /**以下可添加接口请求，接收后端传来的JSON数据，进行画布载入 */

      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          try {
            const jsonData = JSON.parse(e.target.result);
            
            // 检查是否包含必要的数据
            if (jsonData.elements && jsonData.appState) {
              // 更新本地存储
              localStorage.setItem("excalidraw-elements", JSON.stringify(jsonData.elements));
              localStorage.setItem("excalidraw-state", JSON.stringify(jsonData.appState));
              
              if (jsonData.libraries) {
                localStorage.setItem("excalidraw-libs", JSON.stringify(jsonData.libraries));
              }
              
              // 更新线宽滑块值
              if (jsonData.appState.currentItemStrokeWidth) {
                this.strokeWidth = jsonData.appState.currentItemStrokeWidth;
              }
              
              // 重新加载画布
              this.reloadCanvas(jsonData);
              this.$message.success("导入成功!");
            } else {
              this.$message.error("JSON格式无效，缺少必要的画布数据");
            }
          } catch (error) {
            console.error("导入JSON时出错:", error);
            this.$message.error("导入失败: " + error.message);
          }
        };
        reader.readAsText(file);
      }
      
      // 重置文件输入，允许重复选择同一文件
      event.target.value = null;
    },
    
    //重载画布
    reloadCanvas(data) {
      if (excalidrawAPI) {
        // 确保files数据也被加载
        excalidrawAPI.updateScene({
          elements: data.elements,
          appState: data.appState,
          files: data.files || {}
        });
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
.container {
  display: flex;
  height: 100vh;
  background-color: #f5f7fa;
  overflow: hidden;
  position: relative;
  
  .excalidraw {
    flex: 1;
    height: 100%;
    border: 1px solid #ebeef5;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
    background-color: white;
    z-index: 1;
  }
  
  .control-panel {
    width: 200px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    gap: 20px;
    padding: 20px;
    background-color: white;
    border-left: 1px solid #ebeef5;
    box-shadow: -5px 0 15px rgba(0, 0, 0, 0.05);
    z-index: 2;
    
    .action-buttons {
      display: flex;
      flex-direction: column;
      gap: 12px;
      
      .el-button {
        width: 100%;
      }
    }
  }
  
.slider-container {
  span{
    color: #038fe5;
    font-size: 13px;
    font-weight: bold; 
  }
  display: flex;
  flex-direction: column;
  align-items: center;
  background-color: white;
  padding: 8px;
  border-radius: 4px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}
  
  .gallery-content {
    padding: 20px;
    
    .drag-instructions {
      text-align: center;
      margin-bottom: 20px;
      
      .el-tag {
        margin-bottom: 10px;
      }
      
      p {
        color: #606266;
        font-size: 14px;
      }
    }
    
    .gallery-grid {
      display: grid;
      grid-template-columns: repeat(1, 1fr);
      gap: 20px;
    }
    
    .gallery-item {
      border-radius: 8px;
      overflow: hidden;
      box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
      transition: all 0.3s ease;
      background: white;
      cursor: pointer;
      border: 2px solid transparent;
      
      &:hover {
        transform: translateY(-5px);
        box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15);
        border-color: #038fe5;
      }
      
      .image-container {
        height: 150px;
        overflow: hidden;
        
        img {
          width: 100%;
          height: 100%;
          object-fit: cover;
          display: block;
        }
      }
      
      .image-info {
        padding: 12px;
        
        .image-name {
          font-weight: 600;
          color: #303133;
          display: block;
        }
        
        .drag-hint {
          display: block;
          font-size: 12px;
          color: #909399;
          margin-top: 5px;
          
          i {
            margin-right: 4px;
          }
        }
      }
    }
  }

  button {
      padding: 0.3rem 0.8rem;
      background-color: white;
      color: #038fe5;
      border: none;
      border-radius: 4px;
      cursor: pointer;
      font-weight: bold;
      transition: all 0.2s;
      
      &:hover {
        background-color: #f0f0f0;
        transform: translateY(-1px);
      }
      
      &:active {
        transform: translateY(0);
      }
    }
  
  .drop-success-message {
    position: fixed;
    top: 20px;
    left: 50%;
    transform: translateX(-50%);
    background: white;
    padding: 12px 24px;
    border-radius: 8px;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    display: flex;
    align-items: center;
    z-index: 1000;
    border-left: 4px solid #67C23A;
    
    i {
      color: #67C23A;
      font-size: 24px;
      margin-right: 8px;
    }
    
    span {
      color: #303133;
      font-weight: 500;
    }
  }
  
  .loading-overlay {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background-color: rgba(0, 0, 0, 0.5);
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    z-index: 3000;
    backdrop-filter: blur(4px);
    
    .spinner {
      width: 60px;
      height: 60px;
      border: 5px solid rgba(255, 255, 255, 0.3);
      border-radius: 50%;
      border-top: 5px solid #038fe5;
      animation: spin 1s linear infinite;
      margin-bottom: 20px;
    }
    
    p {
      color: white;
      font-size: 1.2rem;
      font-weight: 500;
    }
  }
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.fade-enter-active, .fade-leave-active {
  transition: opacity 0.5s;
}
.fade-enter, .fade-leave-to {
  opacity: 0;
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
  --button-size: 1.8rem !important;
  --icon-size: 1.2rem !important;
}

:global(.excalidraw .App-menu-content) {
  max-height: calc(100vh - 140px);
}

:global(.excalidraw .Island) {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.12) !important;
}

:global(.excalidraw .ToolIcon__icon) {
  opacity: 0.9 !important;
}
</style>