# Forma Frontend Development Plan for Jules Agent
## Single-Agent Execution Strategy for Real-Time Avatar Application

---

## 🎯 Project Overview
Build Forma, a real-time webcam-powered avatar application using the moeru-ai/airi library. This plan is optimized for Jules to execute as a single autonomous agent.

**Target Architecture:**
```
[Webcam] → [Vue.js Frontend] → [WebSocket] → [FastAPI Backend] → [MediaPipe Processing] → [Animation Data] → [Three.js/VRM Renderer]
```

---

## 📋 Pre-Development Setup

### Initial Tasks for Jules:
```bash
# 1. Clone and analyze the reference implementation
git clone https://github.com/moeru-ai/airi.git reference/airi
cd reference/airi && npm install

# 2. Create new project structure
npx create-vite@latest forma-frontend --template vue
cd forma-frontend

# 3. Install all required dependencies upfront
npm install three @pixiv/three-vrm vue-router@4 pinia axios \
  socket.io-client pako lodash-es @stripe/stripe-js \
  @vueuse/core @vueuse/motion floating-vue

# 4. Install dev dependencies
npm install -D @types/three @types/lodash-es sass vitest \
  @vue/test-utils @vitest/ui
```

---

## 🏗️ Phase 1: Foundation & Architecture (Days 1-5)

### Task 1.1: Extract Airi Components
**Location:** `/reference/airi/src/`
**Action:** Analyze and document reusable components
```javascript
// Create: /docs/airi-components.md
// Document these components from airi:
- VRM loader implementation
- Avatar renderer setup  
- Blendshape mapping system
- Animation controller
- Camera controls
```

### Task 1.2: Setup Project Structure
**Create this directory structure:**
```
forma-frontend/
├── src/
│   ├── components/
│   │   ├── avatar/
│   │   ├── camera/
│   │   ├── controls/
│   │   ├── customization/
│   │   └── common/
│   ├── services/
│   │   ├── websocket/
│   │   ├── avatar/
│   │   ├── auth/
│   │   └── payment/
│   ├── composables/
│   ├── stores/
│   ├── views/
│   ├── utils/
│   └── types/
```

### Task 1.3: Core Configuration Files
**Create:** `/src/config/index.ts`
```typescript
export const config = {
  ws: {
    url: process.env.VITE_WS_URL || 'ws://localhost:8000/ws',
    reconnectInterval: 5000,
    maxReconnectAttempts: 10
  },
  camera: {
    width: 640,
    height: 480,
    frameRate: 30,
    facingMode: 'user'
  },
  avatar: {
    defaultModel: '/models/default.vrm',
    renderFPS: 60,
    blendShapeSmoothing: 0.15
  },
  stripe: {
    publicKey: process.env.VITE_STRIPE_PUBLIC_KEY
  }
};
```

### Task 1.4: Setup State Management
**Create:** `/src/stores/index.ts`
```typescript
// Setup Pinia stores for:
- User state (auth, preferences)
- Avatar state (model, customization)
- Connection state (websocket, camera)
- UI state (panels, settings)
```

---

## 🎥 Phase 2: Camera & WebSocket Pipeline (Days 6-10)

### Task 2.1: Webcam Capture Component
**Create:** `/src/components/camera/WebcamCapture.vue`
```vue
<template>
  <div class="webcam-capture">
    <video ref="videoEl" autoplay playsinline />
    <canvas ref="canvasEl" class="hidden" />
    <CameraControls @switch="switchCamera" />
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import { useWebcam } from '@/composables/useWebcam';
import { useWebSocket } from '@/composables/useWebSocket';

const { stream, initCamera, captureFrame } = useWebcam();
const { sendFrame } = useWebSocket();

// Capture and send frames at 30 FPS
const startStreaming = () => {
  setInterval(() => {
    const frame = captureFrame();
    if (frame) sendFrame(frame);
  }, 1000 / 30);
};
</script>
```

### Task 2.2: WebSocket Service
**Create:** `/src/services/websocket/WebSocketManager.ts`
```typescript
export class WebSocketManager {
  private ws: WebSocket | null = null;
  private reconnectTimer: NodeJS.Timeout | null = null;
  private messageQueue: QueuedMessage[] = [];
  
  connect(url: string): void {
    this.ws = new WebSocket(url);
    this.setupEventHandlers();
  }
  
  sendFrame(frameData: Uint8Array): void {
    if (this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(this.encodeFrame(frameData));
    } else {
      this.queueMessage({ type: 'frame', data: frameData });
    }
  }
  
  private handleAnimationData(data: AnimationData): void {
    // Emit to avatar renderer
    eventBus.emit('animation:update', data);
  }
}
```

### Task 2.3: Frame Processing Pipeline
**Create:** `/src/utils/frameProcessor.ts`
```typescript
export class FrameProcessor {
  // Extract frame from video element
  // Compress to JPEG with quality setting
  // Convert to Uint8Array for transmission
  // Add timestamp and metadata
}
```

---

## 🎭 Phase 3: Avatar Rendering System (Days 11-15)

### Task 3.1: Three.js Scene Setup
**Create:** `/src/services/avatar/SceneManager.ts`
```typescript
import * as THREE from 'three';
import { VRM, VRMLoaderPlugin } from '@pixiv/three-vrm';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';

export class SceneManager {
  private scene: THREE.Scene;
  private camera: THREE.PerspectiveCamera;
  private renderer: THREE.WebGLRenderer;
  private vrm: VRM | null = null;
  
  initialize(container: HTMLElement): void {
    // Setup Three.js scene
    // Configure lighting
    // Setup camera position
    // Initialize renderer
  }
  
  async loadVRM(url: string): Promise<void> {
    const loader = new GLTFLoader();
    loader.register((parser) => new VRMLoaderPlugin(parser));
    const gltf = await loader.loadAsync(url);
    this.vrm = gltf.userData.vrm;
    this.scene.add(this.vrm.scene);
  }
  
  applyAnimation(data: AnimationData): void {
    if (!this.vrm) return;
    
    // Apply blendshapes
    Object.entries(data.blendShapes).forEach(([key, value]) => {
      this.vrm.blendShapeProxy?.setValue(key, value);
    });
    
    // Apply bone transforms
    if (data.headRotation) {
      this.vrm.humanoid?.getBone('head')?.rotation.set(...);
    }
  }
}
```

### Task 3.2: Avatar Renderer Component
**Create:** `/src/components/avatar/AvatarRenderer.vue`
```vue
<template>
  <div ref="containerEl" class="avatar-renderer" />
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import { SceneManager } from '@/services/avatar/SceneManager';
import { eventBus } from '@/utils/eventBus';

const containerEl = ref<HTMLElement>();
const sceneManager = new SceneManager();

onMounted(() => {
  sceneManager.initialize(containerEl.value!);
  sceneManager.loadVRM('/models/default.vrm');
  
  // Listen for animation updates
  eventBus.on('animation:update', (data) => {
    sceneManager.applyAnimation(data);
  });
});
</script>
```

### Task 3.3: Animation Synchronization
**Create:** `/src/composables/useAnimationSync.ts`
```typescript
export function useAnimationSync() {
  // Buffer animation frames
  // Interpolate between frames
  // Handle latency compensation
  // Smooth blendshape transitions
}
```

---

## 🎨 Phase 4: Customization System (Days 16-20)

### Task 4.1: Model Selector
**Create:** `/src/components/customization/ModelSelector.vue`
```vue
<template>
  <div class="model-selector">
    <div class="model-grid">
      <ModelCard 
        v-for="model in availableModels"
        :key="model.id"
        :model="model"
        @select="loadModel(model)"
      />
    </div>
    <ImportButton @import="handleModelImport" />
  </div>
</template>
```

### Task 4.2: Customization Panel
**Create:** `/src/components/customization/CustomizationPanel.vue`
```vue
<template>
  <div class="customization-panel">
    <TabGroup>
      <Tab name="Appearance">
        <ColorPicker v-model="colors.hair" label="Hair Color" />
        <ColorPicker v-model="colors.eyes" label="Eye Color" />
        <SliderControl v-model="morphs.faceShape" label="Face Shape" />
      </Tab>
      
      <Tab name="Accessories">
        <AccessoryGrid :items="accessories" @equip="equipAccessory" />
      </Tab>
      
      <Tab name="Animations">
        <AnimationPresets @select="applyAnimation" />
      </Tab>
    </TabGroup>
    
    <PresetManager 
      @save="savePreset"
      @load="loadPreset"
      @export="exportSettings"
    />
  </div>
</template>
```

### Task 4.3: Real-time Preview
**Create:** `/src/composables/useAvatarCustomization.ts`
```typescript
export function useAvatarCustomization() {
  // Watch customization state changes
  // Apply changes to avatar in real-time
  // Handle undo/redo
  // Save/load presets to IndexedDB
}
```

---

## 🎮 Phase 5: User Interface & Controls (Days 21-25)

### Task 5.1: Main Layout
**Create:** `/src/layouts/MainLayout.vue`
```vue
<template>
  <div class="forma-app">
    <aside class="left-panel">
      <WebcamCapture />
      <ConnectionStatus />
    </aside>
    
    <main class="avatar-stage">
      <AvatarRenderer />
      <ControlOverlay />
    </main>
    
    <aside class="right-panel">
      <CustomizationPanel v-if="showCustomization" />
      <SettingsPanel v-if="showSettings" />
    </aside>
    
    <PerformanceMonitor v-if="debug" />
  </div>
</template>
```

### Task 5.2: Control Overlay
**Create:** `/src/components/controls/ControlOverlay.vue`
```vue
<template>
  <div class="control-overlay">
    <div class="quick-actions">
      <button @click="toggleCamera">📷</button>
      <button @click="toggleMic">🎤</button>
      <button @click="screenshot">📸</button>
      <button @click="toggleFullscreen">⛶</button>
    </div>
    
    <div class="view-controls">
      <button @click="resetCamera">Reset View</button>
      <button @click="toggleBackground">BG</button>
    </div>
  </div>
</template>
```

### Task 5.3: Settings Modal
**Create:** `/src/components/controls/SettingsModal.vue`
```vue
<template>
  <Modal v-model="showSettings">
    <SettingsGroup title="Performance">
      <Select v-model="quality" :options="['Low', 'Medium', 'High']" />
      <Switch v-model="antialiasing" label="Antialiasing" />
      <Slider v-model="renderScale" :min="0.5" :max="2" />
    </SettingsGroup>
    
    <SettingsGroup title="Camera">
      <Select v-model="selectedCamera" :options="availableCameras" />
      <Select v-model="resolution" :options="resolutions" />
    </SettingsGroup>
    
    <SettingsGroup title="Keybinds">
      <KeybindEditor />
    </SettingsGroup>
  </Modal>
</template>
```

---

## 🔐 Phase 6: Authentication & Payments (Days 26-30)

### Task 6.1: Authentication Service
**Create:** `/src/services/auth/AuthService.ts`
```typescript
export class AuthService {
  async login(email: string, password: string): Promise<User> {
    const response = await api.post('/auth/login', { email, password });
    this.storeTokens(response.data.tokens);
    return response.data.user;
  }
  
  async loginWithGoogle(): Promise<User> {
    // OAuth flow implementation
  }
  
  async logout(): Promise<void> {
    await api.post('/auth/logout');
    this.clearTokens();
  }
  
  setupAxiosInterceptors(): void {
    // Add auth headers to all requests
    // Handle token refresh
  }
}
```

### Task 6.2: Login/Signup Views
**Create:** `/src/views/auth/LoginView.vue`
```vue
<template>
  <div class="auth-container">
    <form @submit.prevent="handleLogin">
      <Input v-model="email" type="email" placeholder="Email" />
      <Input v-model="password" type="password" placeholder="Password" />
      <Button type="submit">Login</Button>
    </form>
    
    <div class="oauth-options">
      <Button @click="loginWithGoogle">
        <GoogleIcon /> Continue with Google
      </Button>
      <Button @click="loginWithDiscord">
        <DiscordIcon /> Continue with Discord
      </Button>
    </div>
  </div>
</template>
```

### Task 6.3: Stripe Integration
**Create:** `/src/services/payment/StripeService.ts`
```typescript
import { loadStripe } from '@stripe/stripe-js';

export class StripeService {
  private stripe = loadStripe(config.stripe.publicKey);
  
  async createCheckoutSession(priceId: string): Promise<void> {
    const { sessionId } = await api.post('/payments/create-checkout', {
      priceId
    });
    
    const stripe = await this.stripe;
    await stripe.redirectToCheckout({ sessionId });
  }
  
  async manageBilling(): Promise<void> {
    const { url } = await api.post('/payments/billing-portal');
    window.location.href = url;
  }
}
```

---

## 🧪 Phase 7: Testing & Optimization (Days 31-35)

### Task 7.1: Component Tests
**Create:** `/tests/components/WebcamCapture.test.ts`
```typescript
import { mount } from '@vue/test-utils';
import { describe, it, expect } from 'vitest';
import WebcamCapture from '@/components/camera/WebcamCapture.vue';

describe('WebcamCapture', () => {
  it('initializes camera on mount', async () => {
    const wrapper = mount(WebcamCapture);
    await wrapper.vm.$nextTick();
    
    expect(wrapper.vm.stream).toBeTruthy();
    expect(wrapper.find('video').exists()).toBe(true);
  });
  
  it('captures frames at correct rate', async () => {
    // Test 30 FPS capture rate
  });
});
```

### Task 7.2: Performance Optimization
**Create:** `/src/utils/performanceOptimizer.ts`
```typescript
export class PerformanceOptimizer {
  // Implement LOD (Level of Detail) system
  // Add frame rate throttling
  // Optimize texture loading
  // Implement asset caching
  // Add quality auto-adjustment based on performance
}
```

### Task 7.3: Integration Tests
**Create:** `/tests/integration/pipeline.test.ts`
```typescript
describe('End-to-End Pipeline', () => {
  it('streams camera to animated avatar', async () => {
    // Initialize camera
    // Connect WebSocket
    // Load avatar model
    // Verify animation updates
    // Check frame rate
  });
});
```

---

## 🚀 Phase 8: Production Build (Days 36-40)

### Task 8.1: Build Configuration
**Update:** `/vite.config.ts`
```typescript
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor': ['vue', 'pinia', 'vue-router'],
          'three': ['three', '@pixiv/three-vrm'],
          'utils': ['lodash-es', 'axios']
        }
      }
    },
    terserOptions: {
      compress: {
        drop_console: true
      }
    }
  }
});
```

### Task 8.2: Docker Configuration
**Create:** `/Dockerfile`
```dockerfile
FROM node:18-alpine as builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
```

### Task 8.3: CI/CD Pipeline
**Create:** `/.github/workflows/deploy.yml`
```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm run test
      - run: npm run build
      - name: Deploy to Vercel
        run: vercel --prod
```

---

## 📝 Implementation Notes for Jules

### Execution Order:
1. **Start with Phase 1** - Setup and architecture analysis
2. **Build incrementally** - Each phase builds on the previous
3. **Test continuously** - Add tests as you implement features
4. **Integrate early** - Connect components as soon as possible

### Key Files to Create First:
```
1. /src/config/index.ts
2. /src/services/websocket/WebSocketManager.ts
3. /src/components/camera/WebcamCapture.vue
4. /src/services/avatar/SceneManager.ts
5. /src/components/avatar/AvatarRenderer.vue
```

### Testing Checkpoints:
- **After Phase 2:** Camera captures and sends frames
- **After Phase 3:** Avatar loads and receives animation
- **After Phase 4:** Customization changes apply in real-time
- **After Phase 5:** All UI controls functional
- **After Phase 6:** Auth flow complete
- **After Phase 7:** All tests passing

### Performance Targets:
- Camera capture: 30 FPS
- Avatar rendering: 60 FPS
- WebSocket latency: <100ms
- Initial load: <3 seconds
- Memory usage: <500MB

### External Resources Needed:
- Default VRM model file
- MediaPipe models (backend)
- Stripe account (for payments)
- OAuth app credentials

---

## 🎯 Success Criteria

✅ **Phase Completion Checklist:**
- [ ] Webcam streams to backend successfully
- [ ] Avatar animates from webcam data
- [ ] Customization works in real-time
- [ ] User authentication functional
- [ ] Payment integration complete
- [ ] All tests passing (>80% coverage)
- [ ] Production build under 5MB
- [ ] Deployed and accessible

---

## 💡 Jules-Specific Tips

1. **Use the airi repository** as reference for VRM implementation
2. **Start with mock data** if backend isn't ready
3. **Build components in isolation** using Storybook if needed
4. **Comment your code** extensively for future maintenance
5. **Use TypeScript** strictly for better code quality
6. **Implement error boundaries** for robust error handling
7. **Add loading states** for all async operations
8. **Profile performance** regularly with Chrome DevTools

This plan is structured for sequential execution while maintaining the full feature set of the original multi-agent design.
