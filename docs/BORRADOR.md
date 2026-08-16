# tinywasm/iarun

## Librerías y por qué

**Extraíbles (valor propio fuera del motor):**

| Módulo | Rol | Justificación |
|---|---|---|
| `tinywasm/gguf` | Parser de metadata + tensores | Cero dependencias, testeable en Go nativo sin browser. Te sirve también en el server para inspeccionar modelos |
| `tinywasm/tokenizer` | BPE/SentencePiece desde metadata GGUF | Puro, determinista, con vectores de prueba. Reutilizable para contar/truncar tokens |
| `tinywasm/gpu` | Arena de buffers, cache de pipelines y bind groups, dispatch | **El más valioso**: no es un binding crudo sino la capa de gestión encima de `mokiat/wasmgpu`. Te sirve igual para procesar DICOM en el cliente (windowing, filtros, MPR) sin tocar el server |
| `tinywasm/store` | Descarga con HTTP Range + persistencia en OPFS + streaming a GPU | Cargar 2 GB en un browser es *el* problema de UX. Merece su propio módulo y sirve a cualquier asset pesado |

**Internos de `iarun`:**

- `tensor` — dtypes, shapes, strides, bloques cuantizados (Q4_K_M, Q8_0). Sin cómputo: es el contrato entre loader, kernels y grafo.
- `wgsl` — shaders embebidos con `go:embed` + generador de variantes por dtype y tile size (matmul, dequant, rmsnorm, rope, flash-attention). Aquí vive el 80% del rendimiento; separado lo benchmarqueas sin tocar el motor. FlareLLM ronda las cuatro decenas de shaders, para que dimensiones el trabajo.
- `plan` — planificador de grafo y memoria. Crítico: LlamaWeb logra su ventaja con una arena estática de parámetros sin alocación dinámica, buffers intermedios precalculados para FlashAttention, carga por streaming desde OPFS con buffers de transferencia de 1 MB, y un KV-cache de tamaño fijo reservado antes de empezar. [Deciphertech](https://deciphertech.io/blogs/how-browser-llm-inference-became-production-ready-in-2026-what-the-benchmark-data-reveals/) Eso es diseño de memoria, no trucos algorítmicos.
- `arch` — un archivo por arquitectura (llama, qwen, gemma) como composición declarativa de bloques.
- `sampler` — top-k/top-p/min-p, penalties, seed determinista. Puro y testeable.
- `serve` — expone la inferencia por tu endpoint JSON-RPC 2.0 `/mcp`, así el mismo motor alimenta al frontend y al agente.

## Tres decisiones técnicas que definen el proyecto

**1. Go estándar, no TinyGo.** El binario TinyGo pesa menos, pero eso es irrelevante frente a un modelo de 2 GB, y pagas con limitaciones de `reflect`, GC y goroutines justo donde más las necesitas. Guarda TinyGo para módulos chicos.

**2. Minimiza el cruce Go↔JS.** Cada llamada `syscall/js` es cara. Diseña para *un solo command buffer por token*, con bind groups y pipelines pre-creados en el warmup. Si haces una llamada JS por operación, el overhead se come la GPU.

**3. Los pesos nunca tocan la memoria lineal.** wasm32 tope 4 GB, y Safari móvil impone un techo duro de 5 GB por pestaña. [Deciphertech](https://deciphertech.io/blogs/how-browser-llm-inference-became-production-ready-in-2026-what-the-benchmark-data-reveals/) Stream directo desde OPFS a buffers GPU. Además tendrás que shardear tensores grandes por el límite de `maxStorageBufferBindingSize`. Con Q4, el techo realista está entre 3B y 7B parámetros: un 3B a 4 bits ocupa ~1.5–2 GB solo en pesos. [SitePoint](https://www.sitepoint.com/local-first-ai-webgpu-chrome-guide/)

