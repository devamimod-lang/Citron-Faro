# Citron Performance Roadmap

## Diseño actual y razones
Citron emula Nintendo Switch / Tegra X1 con:
- CPU ARM64 vía dynarmic JIT, con page-table fastmem y callbacks de memoria.
- Scheduler Horizon HLE con CoreTiming basado en fibonacci heap y timer thread.
- GPU NVN → Maxwell command stream emulado y recompilado a SPIR-V/Vulkan.
- Memory con CITRON_PAGESIZE 4 KiB, dirty tracking GPU.
- Shader cache y pipeline Vulkan.

Razones de diseño:
- ARM Cortex-A57 requiere JIT rápido; dynarmic aporta fastmem y optimizaciones unsafe opcionales.
- NVN cerrado → recompilar Maxwell a SPIR-V + Vulkan portable.
- CoreTiming asegura sincronía audio/video/GPU en ciclos.
- Fastmem y code-page cache reducen overhead de accesos.

## Puntos de mejora

### 1. CPU dynarmic
**Qué mejorar**
- Habilitar optimizaciones JIT seguras para el host actual: block linking, return-stack buffer, fast dispatcher, context elimination, const prop.
- Ajustar `fastmem_address_space_bits = 64`, `cpuopt_page_tables`, `cpuopt_fastmem`.
- Reducir invalidez de caché de instrucciones: filtrar `InstructionCacheOperationRaised` a rangos realmente ejecutables.

**Archivos**
- src/core/arm/dynarmic/arm_dynarmic_64.cpp
- src/core/arm/dynarmic/arm_dynarmic_64.h

**Métricas**
- IPC del JIT, tiempo de generación de bloque, misses de fastmem.

**Riesgo**
Bajo-medio

### 2. GPU shader recompiler y Vulkan
**Qué mejorar**
- Precarga y persistencia de pipeline cache por juego, warm-up de shaders.
- Reducir cambios de dynamic state en vk_rasterizer, batch draws por pipeline key.
- Usar async GPU `use_async`, turbo mode y frame skipping cuando corresponde.
- Mejorar shader cache `src/video_core/shader_cache.cpp` con invalidación menos agresiva.

**Archivos**
- src/video_core/renderer_vulkan/renderer_vulkan.cpp
- src/video_core/renderer_vulkan/vk_pipeline_cache.cpp
- src/video_core/shader_cache.cpp
- src/video_core/shader_notify.cpp

**Métricas**
- Compilaciones shader/frame, tiempo de submit Vulkan, stutter primera vez.

**Riesgo**
Bajo

### 3. Memoria y coherencia CPU-GPU
**Qué mejorar**
- Ampliar dirty memory manager para evitar barreras innecesarias.
- Alinear lecturas a CITRON_PAGESIZE y batch writes.
- Reducir callbacks OnCPURead/Write en regiones no usadas por GPU.

**Archivos**
- src/core/memory.h
- src/video_core/gpu_dirty_memory_manager.h

**Métricas**
- Overhead de notificaciones CPU-GPU por frame.

**Riesgo**
Medio

### 4. Scheduler y CoreTiming
**Qué mejorar**
- Reducir lock contention en KScopedSchedulerLock.
- Batch eventos de CoreTiming y evitar despertares frecuentes del timer thread.
- Optimizar reschedule cores para evitar busy-wait.

**Archivos**
- src/core/hle/kernel/k_scheduler.h
- src/core/core_timing.h

**Métricas**
- Latencia de cambio de hilo, CPU usage del timer thread.

**Riesgo**
Medio

### 5. Build y optimizaciones
**Qué mejorar**
- Usar IR + CS-IR PGO y LTO full, BOLT para binarios de liberación.
- Habilitar opciones de compilación en docs/BUILDING-CITRON-LINUX.md.

**Archivos**
- build-citron-linux.sh
- docs/BUILDING-CITRON-LINUX.md

**Riesgo**
Bajo

## Próximos pasos
1. Perfilar juego de referencia con CPU y GPU.
2. Aplicar warm-up de pipeline cache.
3. Medir impacto de optimizaciones dynarmic.
