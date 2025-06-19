<template>
  <div class="container" v-loading="loading" element-loading-text="画布加载中...">
    <!-- 画布区域 -->
    <div 
      class="excalidraw" 
      id="excalidraw"
      @dragover.prevent 
      @drop="handleDropOnCanvas"
    ></div>

    <!-- 控制面板 -->
    <div class="control-panel">
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
        <button @click="exportAsJson">导出为可编辑文件</button>
        
        <input 
          type="file" 
          id="jsonImport" 
          accept=".json" 
          @change="importFromJson"
          style="display: none"
        />
        <button @click="triggerImport">导入可编辑文件</button>
        <button @click="exportAsPng" >导出为PNG</button>
        <button @click="exportAsSvg" >导出为SVG</button>
        <button @click="toggleCustomUI">
          {{ showCustomUI ? '简化界面' : '完整界面' }}
        </button>
      </div>
    </div>

    <!-- 图库抽屉 -->
    <div v-show="drawer" class="gallery-content" style="margin-right: 5px;">
      <div class="drag-instructions">
        <span style="color: #038fe5;">点击或拖拽素材添加到画布</span>
      </div>
      
      <div class="gallery-grid">
        <div 
          v-for="(image, index) in galleryImages" 
          :key="image.id" 
          class="gallery-item"
          @click="addImageToCanvas(image.id)"
        >
          <div class="image-container">
            <img :src="base64Images[image.id]" :alt="image.name" />
          </div>
          <div class="image-info">
            <span class="image-name">{{ image.name }}</span>
            <span class="drag-hint">点击添加到画布</span>
          </div>
        </div>
      </div>
    </div>

    <!-- 成功提示 -->
    <div v-if="showDropSuccess" class="drop-success-message">
      <i class="el-icon-success"></i>
      <span>{{ successMessage }}</span>
    </div>
  </div>
</template>

<script>
import { createRoot } from "react-dom/client";
import React from "react";
import { Excalidraw, exportToCanvas, exportToSvg } from "@excalidraw/excalidraw";
import "@excalidraw/excalidraw/index.css";

let root = null;
let excalidrawAPI = null;

// 自定义画笔粗细组件
const StrokeWidthSlider = ({ strokeWidth, onStrokeWidthChange }) => {
  return React.createElement('div', {
    className: 'stroke-width-slider-container',
    style: {
      display: 'flex',
      alignItems: 'center',
      gap: '10px',
      padding: '8px 12px',
      backgroundColor: 'var(--island-bg-color)',
      borderRadius: '8px',
      border: '1px solid var(--default-border-color)',
      boxShadow: '0 2px 8px rgba(0, 0, 0, 0.12)',
      fontSize: '12px',
      color: 'var(--text-color-primary)',
      userSelect: 'none',
      zIndex: 100
    }
  }, [
    React.createElement('span', { 
      key: 'label',
      style: { 
        fontWeight: '500',
        minWidth: '30px'
      }
    }, '线条粗细'),
    React.createElement('input', {
      key: 'slider',
      type: 'range',
      min: 0.1,
      max: 10,
      step: 0.1,
      value: strokeWidth,
      onChange: (e) => onStrokeWidthChange(e.target.value),
      style: {
        width: '80px',
        height: '4px',
        backgroundColor: 'var(--default-border-color)',
        outline: 'none',
        borderRadius: '2px',
        cursor: 'pointer'
      }
    }),
    React.createElement('span', {
      key: 'value',
      style: {
        minWidth: '20px',
        fontWeight: '600',
        color: 'var(--brand-color)'
      }
    }, `${strokeWidth}`)
  ]);
};

export default {
  data() {
    return {
      showCustomUI: false,
      drawer: false,
      strokeWidth: 0.1,
      isLoading: false,
      showDropSuccess: false,
      successMessage: "",
      loading: true,

      galleryImages: [
        { id: 1, name: "山脉", url: "https://images.unsplash.com/photo-1464822759023-fed622ff2c3b?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=600&q=80" },
        { id: 2, name: "海滩", url: "https://images.unsplash.com/photo-1505228395891-9a51e7e86bf6?ixlib=rb-4.0.3&ixid=M3wxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8fA%3D%3D&auto=format&fit=crop&w=600&q=80" },

      ],
      base64Images: {},
      appState: {
        viewBackgroundColor: '#fff',
        currentItemStrokeColor: '#000000',
        currentItemBackgroundColor: '#ffffff',
        currentItemStrokeWidth: 0.1,
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
    this.loading = false;
  },

  beforeDestroy() {
    if (root) {
      root.unmount();
    }
  },

  methods: {
    // 处理画笔粗细变化
    handleStrokeWidthChange(value) {
      this.strokeWidth = value;
      if (excalidrawAPI) {
        excalidrawAPI.updateScene({
          appState: {
            currentItemStrokeWidth: value
          }
        });
      }
    },

    // 导出为PNG
    async exportAsPng() {
      if (!excalidrawAPI) {
        this.$message.error("画布API未初始化");
        return;
      }

      try {
        const elements = excalidrawAPI.getSceneElements();
        const appState = excalidrawAPI.getAppState();
        const files = excalidrawAPI.getFiles();

        if (elements.length === 0) {
          this.$message.warning("画布为空，无法导出");
          return;
        }

        const canvas = await exportToCanvas({
          elements,
          appState,
          files,
          exportPadding: 20,
        });

        const fileName = `drawing-${new Date().toISOString().slice(0, 19).replace(/:/g, '-')}.png`;
        
        canvas.toBlob((blob) => {
          const url = URL.createObjectURL(blob);
          const link = document.createElement('a');
          link.href = url;
          link.download = fileName;
          document.body.appendChild(link);
          link.click();
          document.body.removeChild(link);
          URL.revokeObjectURL(url);
          
          this.$message.success("PNG导出成功!");
        }, 'image/png');

      } catch (error) {
        console.error("导出PNG时出错:", error);
        this.$message.error(`导出PNG失败: ${error.message}`);
      }
    },

    // 导出为SVG
    async exportAsSvg() {
      if (!excalidrawAPI) {
        this.$message.error("画布API未初始化");
        return;
      }

      try {
        const elements = excalidrawAPI.getSceneElements();
        const appState = excalidrawAPI.getAppState();
        const files = excalidrawAPI.getFiles();

        if (elements.length === 0) {
          this.$message.warning("画布为空，无法导出");
          return;
        }

        const svg = await exportToSvg({
          elements,
          appState,
          files,
          exportPadding: 20,
        });

        const fileName = `drawing-${new Date().toISOString().slice(0, 19).replace(/:/g, '-')}.svg`;
        const svgData = new XMLSerializer().serializeToString(svg);
        const blob = new Blob([svgData], { type: 'image/svg+xml' });
        const url = URL.createObjectURL(blob);
        
        const link = document.createElement('a');
        link.href = url;
        link.download = fileName;
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        URL.revokeObjectURL(url);
        
        this.$message.success("SVG导出成功!");

      } catch (error) {
        console.error("导出SVG时出错:", error);
        this.$message.error(`导出SVG失败: ${error.message}`);
      }
    },

    // 优化后的图片添加方法
    async addImageToCanvas(imageId) {
      console.log("===== 开始添加图片到画布 =====");
      console.log("图片ID:", imageId);
      
      if (!excalidrawAPI) {
        console.error("excalidrawAPI 未初始化");
        this.$message.error("画布API未初始化");
        return;
      }
      
      try {
        const base64Data = this.base64Images[imageId];
        if (!base64Data) {
          console.error("未找到图片数据，imageId:", imageId);
          this.$message.error("未找到图片数据");
          return;
        }
        
        const fileId = `image-${imageId}-${Date.now()}`;
        const elementId = `element-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
        
        const fileData = {
          id: fileId,
          dataURL: base64Data,
          mimeType: this.getMimeTypeFromBase64(base64Data),
          created: Date.now()
        };
        
        await excalidrawAPI.addFiles([fileData]);
        
        const imageElement = {
          type: "image",
          id: elementId,
          x: 200,
          y: 200,
          width: 200,
          height: 200,
          angle: 0,
          strokeColor: "transparent",
          backgroundColor: "transparent",
          fillStyle: "solid",
          strokeWidth: 0.1,
          strokeStyle: "solid",
          roughness: 1,
          opacity: 100,
          groupIds: [],
          frameId: null,
          roundness: null,
          seed: Math.floor(Math.random() * 2147483648),
          version: 1,
          versionNonce: Math.floor(Math.random() * 2147483648),
          isDeleted: false,
          boundElements: null,
          updated: Date.now(),
          link: null,
          locked: false,
          fileId: fileId,
          scale: [1, 1]
        };
        
        const currentElements = excalidrawAPI.getSceneElements();
        const newElements = [...currentElements, imageElement];
        
        excalidrawAPI.updateScene({
          elements: newElements
        });
        
        this.saveWithFiles();
        this.showSuccessMessage("图片添加成功");
        
      } catch (error) {
        console.error("添加图片到画布时出错:", error);
        this.$message.error(`添加图片失败: ${error.message}`);
      }
    },

    // 新增：保存包含文件数据的方法
    saveWithFiles() {
      if (!excalidrawAPI) return;
      
      const appState = excalidrawAPI.getAppState();
      const elements = excalidrawAPI.getSceneElements();
      const files = excalidrawAPI.getFiles();
      
      const { collaborators, ...stateToSave } = appState;
      stateToSave.currentItemStrokeWidth = this.strokeWidth;
      
      localStorage.setItem("excalidraw-state", JSON.stringify(stateToSave));
      localStorage.setItem("excalidraw-elements", JSON.stringify(elements));
      localStorage.setItem("excalidraw-files", JSON.stringify(files));
      localStorage.setItem("excalidraw-base64Images", JSON.stringify(this.base64Images));
    },

    // 从base64数据中提取MIME类型
    getMimeTypeFromBase64(base64String) {
      const matches = base64String.match(/^data:([a-zA-Z0-9]+\/[a-zA-Z0-9-.+]+);base64,/);
      if (matches && matches.length > 1) {
        return matches[1];
      }
      return 'image/jpeg';
    },

    // 优化后的图片加载方法
    async loadImages() {
      console.log("===== 开始加载图片 =====");
      this.isLoading = true;
      
      const savedBase64Images = localStorage.getItem("excalidraw-base64Images");
      if (savedBase64Images) {
        try {
          this.base64Images = JSON.parse(savedBase64Images);
          console.log("从localStorage恢复图片数据成功");
          this.isLoading = false;
          return;
        } catch (error) {
          console.warn("localStorage中的图片数据无效，重新加载");
        }
      }
      
      const imagesMap = {};
      
      try {
        for (const [index, image] of this.galleryImages.entries()) {
          console.log(`正在加载第 ${index + 1} 张图片:`, image.name);
          
          try {
            const controller = new AbortController();
            const timeoutId = setTimeout(() => controller.abort(), 10000);
            
            const response = await fetch(image.url, {
              signal: controller.signal,
              mode: 'cors'
            });
            clearTimeout(timeoutId);
            
            if (!response.ok) {
              throw new Error(`HTTP错误: ${response.status}`);
            }
            
            const blob = await response.blob();
            const base64 = await this.convertBlobToBase64(blob);
            
            imagesMap[image.id] = base64;
            console.log(`图片 ${image.name} 加载成功`);
            
          } catch (imageError) {
            console.error(`图片 ${image.name} 加载失败:`, imageError);
          }
        }
        
        this.base64Images = imagesMap;
        localStorage.setItem("excalidraw-base64Images", JSON.stringify(imagesMap));
        console.log("图片加载完成并保存到localStorage");
        
      } catch (err) {
        console.error("图片批量加载失败:", err);
        this.$message.error("图片加载失败，请检查网络连接");
      } finally {
        this.isLoading = false;
      }
    },

    // base64转换方法
    convertBlobToBase64(blob) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onloadend = () => resolve(reader.result);
        reader.onerror = reject;
        reader.readAsDataURL(blob);
      });
    },

    // 优化后的初始化方法 - 关键修改
    initializeExcalidraw() {
      const elements = JSON.parse(localStorage.getItem("excalidraw-elements") || "[]");
      console.log("从localStorage加载的元素:", elements);
      const libs = JSON.parse(localStorage.getItem("excalidraw-libs") || "[]");
      const savedState = JSON.parse(localStorage.getItem("excalidraw-state") || "{}");
      const savedFiles = JSON.parse(localStorage.getItem("excalidraw-files") || "{}");
      
      if (savedState.currentItemStrokeWidth) {
        this.strokeWidth = savedState.currentItemStrokeWidth;
      }
      
      const initialState = { ...this.appState, ...savedState };
      
      root = createRoot(document.getElementById("excalidraw"));
      root.render(
        React.createElement(Excalidraw, {
          name: "我的画板",
          initialData: {
            elements: elements,
            libraryItems: libs,
            appState: initialState,
            files: savedFiles
          },
          langCode: "zh-CN",
          onChange: this.onChange,
          onLibraryChange: this.onLibraryChange,
          UIOptions: this.appState.uiOptions,
          excalidrawAPI: (api) => {
            excalidrawAPI = api;
            window.excalidrawAPI = api;
          },
          // 添加自定义UI组件
          renderTopRightUI: () => {
            return React.createElement(StrokeWidthSlider, {
              strokeWidth: this.strokeWidth,
              onStrokeWidthChange: this.handleStrokeWidthChange
            });
          }
        })
      );
    },

    // 显示成功消息
    showSuccessMessage(message) {
      this.successMessage = message;
      this.showDropSuccess = true;
      
      setTimeout(() => {
        this.showDropSuccess = false;
      }, 3000);
    },

    // 更新线条粗细
    updateStrokeWidth(value) {
      if (!excalidrawAPI) return;
      
      excalidrawAPI.updateScene({
        appState: {
          currentItemStrokeWidth: value
        }
      });
    },
    
    // 应用自定义CSS
    applyCustomCSS() {
      const styleEl = document.createElement('style');
      styleEl.id = 'excalidraw-custom-styles';
      this.updateCustomCSS(styleEl);
      document.head.appendChild(styleEl);
    },
    
    // 更新自定义CSS
    updateCustomCSS(styleEl = document.getElementById('excalidraw-custom-styles')) {
      if (!styleEl) return;
      
      if (this.showCustomUI) {
        styleEl.textContent = `
          /* 添加画笔粗细滑动条的样式 */
          .stroke-width-slider-container input[type="range"] {
            -webkit-appearance: none;
            appearance: none;
            background: transparent;
            cursor: pointer;
          }
          
          .stroke-width-slider-container input[type="range"]::-webkit-slider-track {
            background: var(--default-border-color);
            height: 4px;
            border-radius: 2px;
          }
          
          .stroke-width-slider-container input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none;
            appearance: none;
            background: var(--brand-color);
            height: 16px;
            width: 16px;
            border-radius: 50%;
            cursor: pointer;
            border: 2px solid white;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
          }
          
          .stroke-width-slider-container input[type="range"]::-moz-range-track {
            background: var(--default-border-color);
            height: 4px;
            border-radius: 2px;
            border: none;
          }
          
          .stroke-width-slider-container input[type="range"]::-moz-range-thumb {
            background: var(--brand-color);
            height: 16px;
            width: 16px;
            border-radius: 50%;
            cursor: pointer;
            border: 2px solid white;
            box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
          }
        `;
        return;
      }
      
      styleEl.textContent = `
        .layer-ui__library {
          display: none !important;
        }
        .CollabButton {
          display: none !important;
        }
        .dropdown-menu-group-sessions,
        .dropdown-menu-item-export,
        .dropdown-menu-item-save,
        .dropdown-menu-item-load {
          display: none !important;
        }
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

        button[data-testid="main-menu-trigger"][type="button"]{
          display: none !important;
        }

        button[data-testid="search-menu-button"][type="button"]{
          display: none !important;
        }
        .panel-row-horizontal > .ToolIcon[title="锁定"] {
          display: none !important;
        }
        
        /* 添加画笔粗细滑动条的样式 */
        .stroke-width-slider-container input[type="range"] {
          -webkit-appearance: none;
          appearance: none;
          background: transparent;
          cursor: pointer;
        }
        
        .stroke-width-slider-container input[type="range"]::-webkit-slider-track {
          background: var(--default-border-color);
          height: 4px;
          border-radius: 2px;
        }
        
        .stroke-width-slider-container input[type="range"]::-webkit-slider-thumb {
          -webkit-appearance: none;
          appearance: none;
          background: var(--brand-color);
          height: 16px;
          width: 16px;
          border-radius: 50%;
          cursor: pointer;
          border: 2px solid white;
          box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
        }
        
        .stroke-width-slider-container input[type="range"]::-moz-range-track {
          background: var(--default-border-color);
          height: 4px;
          border-radius: 2px;
          border: none;
        }
        
        .stroke-width-slider-container input[type="range"]::-moz-range-thumb {
          background: var(--brand-color);
          height: 16px;
          width: 16px;
          border-radius: 50%;
          cursor: pointer;
          border: 2px solid white;
          box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
        }
      `;
    },
    
    // 切换UI显示模式
    toggleCustomUI() {
      this.showCustomUI = !this.showCustomUI;
      this.updateCustomCSS();
    },
    
    // 优化后的保存方法
    save() {
      this.saveWithFiles();
      this.$message.success("保存成功!");
    },
    
    // 导出JSON
    exportAsJson() {
      if (excalidrawAPI) {
        const appState = excalidrawAPI.getAppState();
        const elements = excalidrawAPI.getSceneElements();
        const files = excalidrawAPI.getFiles();
        
        const { collaborators, ...stateToSave } = appState;
        stateToSave.currentItemStrokeWidth = this.strokeWidth;
        
        const exportData = {
          elements: elements,
          appState: stateToSave,
          libraries: JSON.parse(localStorage.getItem("excalidraw-libs") || "[]"),
          files: files || {},
          base64Images: this.base64Images
        };
        
        const dataStr = JSON.stringify(exportData, null, 2);
        const dataUri = "data:application/json;charset=utf-8," + encodeURIComponent(dataStr);
        const exportFileName = `drawing-${new Date().toISOString().slice(0, 10)}.json`;
        
        const linkElement = document.createElement("a");
        linkElement.setAttribute("href", dataUri);
        linkElement.setAttribute("download", exportFileName);
        linkElement.click();
      }
    },
    
    // 触发导入
    triggerImport() {
      document.getElementById("jsonImport").click();
    },
    
    // 优化后的导入方法
    importFromJson(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          try {
            const jsonData = JSON.parse(e.target.result);
            
            if (jsonData.elements && jsonData.appState) {
              localStorage.setItem("excalidraw-elements", JSON.stringify(jsonData.elements));
              localStorage.setItem("excalidraw-state", JSON.stringify(jsonData.appState));
              
              if (jsonData.libraries) {
                localStorage.setItem("excalidraw-libs", JSON.stringify(jsonData.libraries));
              }
              
              if (jsonData.files) {
                localStorage.setItem("excalidraw-files", JSON.stringify(jsonData.files));
              }
              
              if (jsonData.base64Images) {
                localStorage.setItem("excalidraw-base64Images", JSON.stringify(jsonData.base64Images));
                this.base64Images = jsonData.base64Images;
              }
              
              if (jsonData.appState.currentItemStrokeWidth) {
                this.strokeWidth = jsonData.appState.currentItemStrokeWidth;
              }
              
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
      
      event.target.value = null;
    },
    
    // 重载画布
    reloadCanvas(data) {
      if (excalidrawAPI) {
        excalidrawAPI.updateScene({
          elements: data.elements,
          appState: data.appState,
          files: data.files || {}
        });
      }
    },
    
    // onChange事件处理
    onChange(elements, appState, files) {
      localStorage.setItem("excalidraw-elements", JSON.stringify(elements));
      
      if (files) {
        localStorage.setItem("excalidraw-files", JSON.stringify(files));
      }
    },
    
    // 库变化处理
    onLibraryChange(libraryItems) {
      localStorage.setItem("excalidraw-libs", JSON.stringify(libraryItems));
    }
  }
};
</script>

<style scoped lang="scss">
.container {
  display: flex;
  flex-direction: row;
  height: 100vh;
  background-color: #f5f7fa;
  overflow: hidden;
  position: relative;
  
  .excalidraw {
    flex: 1;
    height: 100vh;
    border: 1px solid #ebeef5;
    background-color: white;
    z-index: 1;
  }
  
  .control-panel {
    width: 200px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    gap: 20px;
    
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
      
      .export-button {
        padding: 0.5rem 1rem;
        border-radius: 6px;
        font-weight: 600;
        font-size: 13px;
        transition: all 0.3s ease;
        
        &.png-button {
          background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
          color: white;
          
          &:hover {
            background: linear-gradient(135deg, #5a6fd8 0%, #6a4190 100%);
            transform: translateY(-2px);
            box-shadow: 0 4px 15px rgba(102, 126, 234, 0.4);
          }
        }
        
        &.svg-button {
          background: linear-gradient(135deg, #f093fb 0%, #f5576c 100%);
          color: white;
          
          &:hover {
            background: linear-gradient(135deg, #ee82e9 0%, #f3455a 100%);
            transform: translateY(-2px);
            box-shadow: 0 4px 15px rgba(240, 147, 251, 0.4);
          }
        }
        
        &:active {
          transform: translateY(0);
        }
      }
    }
  }
  
  .gallery-content {
    top: 0;
    right: 200px;
    bottom: 0;
    width: 240px;
    background: white;
    box-shadow: -5px 0 15px rgba(0, 0, 0, 0.1);
    overflow-y: auto;
    z-index: 2;
    margin-right: 20px;
    
    .drag-instructions {
      text-align: center;
      margin: 15px 0;
      
      span {
        font-size: 14px;
        font-weight: 500;
      }
    }
    
    .gallery-grid {
      display: grid;
      grid-template-columns: repeat(1, 1fr);
      gap: 20px;
      padding: 10px;
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
    top: 80px;
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
}

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