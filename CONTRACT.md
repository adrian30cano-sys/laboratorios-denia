# CONTRACT.md · Laboratorios Dénia · Módulo 3D (HumanRig)

Interfaz fija entre los 4 bloques de construcción del módulo 3D (Fase 3) y entre el módulo 3D y el resto de la web (Fase 6-7). **No se renombra nada de aquí sin actualizar este archivo y avisar en el registro de decisiones de CEREBRO.**

---

## 1. Jerarquía de joints (nombres exactos, `THREE.Group`)

```
root
└─ pelvis
   └─ spine_lumbar
      └─ spine_thoracic
         ├─ neck
         │  └─ head
         │     ├─ jaw
         │     ├─ eye_L
         │     └─ eye_R
         ├─ clavicle_L → shoulder_L → elbow_L → wrist_L
         │     └─ mano_L: thumb_cmc_L, thumb_mcp_L, thumb_ip_L,
         │                index_mcp_L, index_pip_L, index_dip_L,
         │                middle_mcp_L, middle_pip_L, middle_dip_L,
         │                ring_mcp_L, ring_pip_L, ring_dip_L,
         │                pinky_mcp_L, pinky_pip_L, pinky_dip_L
         └─ clavicle_R → shoulder_R → elbow_R → wrist_R
               └─ mano_R: (mismo esquema que mano_L, sufijo _R)
   ├─ hip_L → knee_L → ankle_L → toe_L
   └─ hip_R → knee_R → ankle_R → toe_R
```

- Sufijo `_L` / `_R` en todo par bilateral. Sin excepciones ni abreviaturas alternativas.
- Cada dedo expone un `curl` (0–1) individual; función global `grasp(0–1)` que mueve los 5 dedos de una mano a la vez.

## 2. Proporciones (ALTURA_TOTAL = 1.75, deben sumar ≈1.75 con los pies en y=0)

| Segmento | Longitud |
|---|---|
| Pie (alto) | 0.08 |
| Tibia | 0.41 |
| Fémur | 0.43 |
| Pelvis → y centro | 0.95 |
| Columna lumbar | 0.20 |
| Columna torácica | 0.28 |
| Cuello | 0.08 |
| Cabeza | 0.23 |
| Húmero | 0.30 |
| Antebrazo | 0.26 |
| Mano | 0.19 |

Ancho biacromial 0.40 · ancho de cadera (centros articulares) 0.19 · pie 0.26 largo × 0.08 alto.

## 3. Zonas de raycast (`userData.zone`, valores exactos)

| Valor de `userData.zone` | Mallas que lo llevan | Acción de clic |
|---|---|---|
| `head` | head, jaw, eye_L, eye_R | Saludo: asentimiento + gesto de mano |
| `chest` | pecho (spine_thoracic, frontal) | Pulso de latido (escala + emissive) + panel |
| `shoulder_arm` | clavicle, shoulder, elbow, wrist (ambos lados) | Elevación y flexión del brazo con easing |
| `hand` | mano completa (ambos lados) | Alternar grasp abrir/cerrar + saludo |
| `abdomen_back` | spine_lumbar, pelvis (frontal y espalda) | Giro de tronco; en espalda, giro 180° |
| `leg` | hip, knee, ankle (ambos lados) | Elevación de rodilla / paso |
| `foot` | ankle, toe (ambos lados) | Flexión plantar/dorsal |

Toda malla sin zona explícita no es interactiva (no lleva `userData.zone`).

## 4. Límites de movimiento (grados, clamp obligatorio en cada joint)

| Joint | Límites |
|---|---|
| neck | flex/ext −40/+50 · rot ±70 · lateral ±35 |
| spine (lumbar+torácica, total) | flex 40 · ext 20 · lateral ±20 · rot ±35 |
| shoulder | flex 0–180 · ext −50 · abd 0–170 · rot ±80 |
| elbow | 0–145 |
| wrist | flex/ext ±70 · desviación ±25 |
| hip | flex 0–120 · ext −20 · abd −40/+45 |
| knee | 0–135 |
| ankle | dorsiflexión 20 · plantarflexión 40 |
| falanges (MCP/PIP) | 0–90 |
| falanges (DIP) | 0–70 |

## 5. API pública — `window.humanRig` (firmas exactas)

```ts
interface ZoneClickPayload {
  zone: 'head' | 'chest' | 'shoulder_arm' | 'hand' | 'abdomen_back' | 'leg' | 'foot';
  side: 'L' | 'R' | null;       // null si la zona no es bilateral (head, chest, abdomen_back)
  point: { x: number; y: number; z: number }; // coordenada mundial del impacto
}

interface ZoneHoverPayload {
  zone: ZoneClickPayload['zone'] | null; // null cuando el hover termina
  side: 'L' | 'R' | null;
}

class HumanRig {
  mount(container: HTMLElement): void;
  dispose(): void; // libera geometrías, materiales, listeners y para el render loop

  setPose(nombreOPose: string | Record<string, {x:number;y:number;z:number}>): void;
  playAnimation(nombre: string): void;
  setJoint(jointName: string, rot: {x:number;y:number;z:number}): void; // clamp automático a la sección 4
  highlightZone(zona: ZoneClickPayload['zone']): void;
  setLayerMode(modo: 'skin' | 'xray'): void;
  reset(): void;

  on(evento: 'zoneClick', callback: (payload: ZoneClickPayload) => void): void;
  on(evento: 'zoneHover', callback: (payload: ZoneHoverPayload) => void): void;
}
```

- `mount()`/`dispose()` deben soportar ciclos repetidos (React StrictMode monta dos veces en desarrollo) sin fugas: tras 10 ciclos, `renderer.info.memory.geometries` y `.textures` deben volver a 0.
- `on()` devuelve una función de desuscripción implícita al llamar `dispose()`; no hace falta un `off()` explícito en la v1.

## 6. Contenido por zona (JSON editable, no texto embebido en el código)

Estructura que consumirá `ZonePanel` — el módulo 3D solo emite `zone`/`side`, nunca el texto:

```json
{
  "head": { "nombre_anatomico": "", "texto": "", "dato_curioso": "" },
  "chest": { "nombre_anatomico": "", "texto": "", "dato_curioso": "" },
  "abdomen_back": { "nombre_anatomico": "", "texto": "", "dato_curioso": "" },
  "shoulder_arm": { "nombre_anatomico": "", "texto": "", "dato_curioso": "" },
  "hand": { "nombre_anatomico": "", "texto": "", "dato_curioso": "" },
  "leg": { "nombre_anatomico": "", "texto": "", "dato_curioso": "" },
  "foot": { "nombre_anatomico": "", "texto": "", "dato_curioso": "" }
}
```

Vive en `web/src/content/zones.es.json` y `zones.en.json`. Vacío hasta que el cliente lo aporte (ver Fase 1 del proyecto).

## 7. Restricciones heredadas del prompt original

- Solo `three@0.128.0` (mismo build que r128 UMD). Sin `CapsuleGeometry` ni `OrbitControls`.
- Sin `localStorage`, `sessionStorage` ni `fetch` dentro de `lib/human-rig/`.
- `dispose()` es obligatorio y debe ser exhaustivo: geometrías, materiales, texturas, listeners de puntero/teclado, `ResizeObserver`, render loop.

---
*Cualquier cambio a este archivo se registra en la ficha de CEREBRO del proyecto antes de tocar código.*
