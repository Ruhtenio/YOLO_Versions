# StockEye v4 · YOLO26 — Detección de góndolas y espacios vacíos

Modelo YOLO26 entrenado para detectar **góndolas** y **espacios vacíos (quiebres de stock)** en fotos de supermercado, entrenado completamente en CPU.

## Contenido de la carpeta

| Archivo | Qué es |
|---|---|
| `StockEye_v4_calidad.ipynb` | Notebook completo: preparación de fotos, etiquetado asistido, revisión humana, entrenamiento, evaluación y análisis de quiebres. Incluye las salidas de la última ejecución. |
| `dataset_gondolas_2026-10-13.zip` | Dataset en formato YOLO con las 187 fotos y sus cajas revisadas a mano. |

### Estructura del dataset

```
dataset_gondolas_2026-10-13/
├── images/       187 fotos .jpg
├── labels/       187 .txt  (clase xc yc w h, normalizado 0–1)
├── classes.txt   gondola, vacio
└── data.yaml     0: gondola · 1: vacio
```

| Clase | Cajas | Tamaño mediano (relativo a la foto) | Equivalente a 640 px |
|---|---|---|---|
| `0 gondola` | 324 | 50 % × 62 % | ≈ 323 × 400 px |
| `1 vacio` | 1288 | 4,4 % × 5,5 % | ≈ 28 × 35 px |

> El `data.yaml` del zip apunta `train` y `val` a la misma carpeta para que se pueda usar tal cual. Para entrenar conviene separarlos. El notebook lo hace automáticamente (celda 4), agrupando ráfagas de fotos para que no haya fugas entre train y val.

## Metodología

1. **Fotos:** 187 fotos de góndolas (cámara nativa y WhatsApp), reducidas y normalizadas.
2. **Etiquetado automático:** un ensemble de detectores (YOLOv8 de foduu, modelo de góndolas de Colab, vacíos sugeridos por geometría) propone cajas iniciales.
3. **Revisión humana:** las **187 fotos se revisaron y corrigieron a mano** en un editor interactivo dentro del notebook.
4. **Split sin fugas:** 148 fotos de train y 39 de validación.
5. **Entrenamiento en dos rondas (maestro → final):**
   - **Ronda 1 (maestro):** 25 épocas solo con fotos revisadas. Como no quedaron fotos pendientes, el maestro no tuvo nada nuevo que proponer.
   - **Ronda final:** hasta 50 épocas con *early stopping* (paciencia 12).

### Configuración

| Parámetro | Valor |
|---|---|
| Modelo | YOLO26s (9,47 M parámetros, 20,8 GFLOPs) |
| Perfil | `equilibrado` |
| Tamaño de imagen | 640 px |
| Batch | 8 |
| Aumentos | flip horizontal, ±5° de rotación, shear, escala 0,5, variaciones de color y luz |
| Hardware | CPU AMD Ryzen 5 3400G (sin GPU), Ultralytics 8.4.176, PyTorch 2.14.1 |

## Resultados

### Entrenamiento

| Ronda | Épocas | Tiempo | mAP50 | mAP50-95 |
|---|---|---|---|---|
| Maestro | 25 / 25 | 53 min | 0,540 | 0,281 |
| **Final** | **37 / 50** (early stopping, mejor época: 25) | 73 min | **0,544** | **0,304** |

La CPU llegó como máximo a 70 °C. La inferencia toma unos **150 ms por imagen** en CPU.

### Evaluación por clase (validación: 39 fotos, 315 cajas)

| Clase | Precisión | Recall | mAP50 | mAP50-95 |
|---|---|---|---|---|
| **gondola** | 0,715 | **0,824** | **0,862** | 0,542 |
| **vacio** | 0,559 | **0,178** | **0,226** | 0,066 |
| Total | 0,637 | 0,501 | 0,544 | 0,304 |

### ¿Encuentra cada vacío? (celda 22, umbral de operación)

| Métrica | Valor |
|---|---|
| Vacíos reales en validación | 263 |
| Encontrados | 69 |
| Se escaparon | 194 |
| Falsas alarmas | 12 |
| **Recall de vacíos** | **0,26** |
| **Precisión de vacíos** | **0,85** |

### Análisis de quiebres por góndola (celda 23)

De **61 góndolas** detectadas en validación, el modelo levantó **1 alerta** de quiebre de stock.

## Conclusión

**El modelo detecta góndolas bien, pero todavía no sirve para detectar quiebres de stock de forma confiable.**

- **Góndolas: buen resultado.** Con mAP50 0,86 y recall 0,82, el modelo encuentra la gran mayoría de las góndolas con un dataset de solo 148 fotos de entrenamiento. Esta parte ya es utilizable para localizar las góndolas en una foto.
- **Vacíos: preciso pero muy conservador.** Cuando marca un vacío, acierta el 85 % de las veces y casi no da falsas alarmas (12). Pero **solo encuentra 1 de cada 4 vacíos reales** (194 de 263 se escapan). Por eso el análisis por góndola casi no levanta alertas (1 de 61): el problema no es que las góndolas estén llenas, es que el modelo no ve la mayoría de los huecos.
- **La causa principal es el tamaño de los vacíos.** Un vacío mide en promedio unos 28 × 35 px a 640 px, muy pequeño para el modelo después de reducir la foto. Las góndolas, en cambio, ocupan media imagen, y por eso se detectan bien.
- **El entrenamiento se estancó pronto.** La mejor época fue la 25 y las 12 siguientes no mejoraron. Con estos datos y esta resolución, más épocas no ayudan: hacen falta más información (resolución) o más ejemplos.

### Próximos pasos recomendados

1. **Subir la resolución:** entrenar con el perfil `calidad` (960 px) o mayor, para que los vacíos tengan más píxeles.
2. **Inferencia por mosaico (tiles/SAHI):** dividir cada foto en partes y detectar en cada una. El notebook ya tiene la opción `MOSAICO_INFERENCIA` en la celda 23.
3. **Bajar el umbral de confianza para la clase vacío:** como la precisión es alta (0,85), hay margen para aceptar más detecciones y ganar recall.
4. **Más fotos y un criterio de etiquetado uniforme:** más ejemplos de vacíos, sobre todo de distintos tamaños, y una regla clara de qué cuenta como "vacío".
5. **Usar el maestro con fotos nuevas:** cuando se agreguen fotos sin etiquetar, el maestro puede proponer las cajas y acelerar la revisión.
