# U1 - Proyecto Integrador: Escenario Procedural con Animación de Cámara

---

## 📋 Descripción del Proyecto

Este proyecto consiste en la **generación procedural de un escenario 3D** dentro de **Blender** utilizando su API de Python (`bpy`). El escenario es un pasillo curvo con materiales aplicados, suelo generado por malla y una **cámara animada** que recorre el pasillo de principio a fin.

El proyecto está basado en la tarea *Escenario Procedural*, a la que se le agregó la funcionalidad principal de **animación de la cámara a través del camino (path animation)**.

---

## Características del proyecto:

- Generación **100% procedural** del escenario (sin modelado manual)
- Pasillo con **curvatura dinámica** calculada matemáticamente
- Materiales creados con nodos (**Principled BSDF**)
- Suelo generado con una **malla continua** siguiendo la curva del pasillo
- **Cámara animada** con keyframes que sigue el recorrido del pasillo
- Interpolación **Bezier** para un movimiento de cámara suave y fluido

---

##  Explicación detallada del código

### 1. Creación de Materiales

```python
def crear_material(nombre, color_rgb):
    mat = bpy.data.materials.new(name=nombre)
    mat.use_nodes = True
    bsdf = mat.node_tree.nodes.get("Principled BSDF")
    if bsdf:
        bsdf.inputs["Base Color"].default_value = (*color_rgb, 1.0)
    return mat
```

Se crean 3 materiales:
- `ParedOscura` → color casi negro `(0.1, 0.1, 0.1)`
- `ParedDetalle` → color naranja-rojo `(0.8, 0.2, 0.0)`
- `Suelo` → color gris oscuro `(0.15, 0.15, 0.15)`

<img width="686" height="356" alt="Captura5" src="https://github.com/user-attachments/assets/7815e12c-ed23-4b68-80de-a1aa965ff61e" />  
<img width="1062" height="555" alt="Captura6" src="https://github.com/user-attachments/assets/1c38be14-e130-409d-b993-9270a552876b" />

---

### 2. Curvatura del Pasillo (`offset_x`)

```python
def offset_x(i):
    x = 0.0
    if 15 <= i <= 30:
        t = (i - 15) / 15.0
        x += 6.0 * (0.5 - 0.5 * math.cos(t * math.pi))
    elif i > 30:
        x += 6.0
    if 38 <= i <= 53:
        t = (i - 38) / 15.0
        x -= 6.0 * (0.5 - 0.5 * math.cos(t * math.pi))
    elif i > 53:
        x -= 6.0
    return x
```

Usa una función **coseno suavizada** para generar curvas graduales. El pasillo se desplaza `+6` unidades en X entre los bloques 15–30, y luego regresa `-6` unidades entre los bloques 38–53, creando un recorrido en forma de S.


<img width="1047" height="544" alt="Captura7" src="https://github.com/user-attachments/assets/5783a38e-29d4-4cc3-87ba-339b4178647b" />


---

### 3. Ángulo de Tangente (`angulo_tangente`)

```python
def angulo_tangente(i):
    dx = offset_x(min(i + 1, total_bloques - 1)) - offset_x(max(i - 1, 0))
    dy = paso * 2
    return math.atan2(dx, dy)
```

Calcula la **dirección de avance** del pasillo en cada bloque usando diferencias finitas, de modo que las paredes y la cámara siempre apunten en la dirección correcta del recorrido.

---

### 4. Generación de Paredes

Se generan **60 bloques** de paredes (izquierda y derecha) usando cubos primitivos:

```python
for i in range(total_bloques):
    # Pared izquierda con alternancia de material y escala
    bpy.ops.mesh.primitive_cube_add(location=(cx - ancho, cy, altura_pared / 2))
    ...
    # Pared derecha
    bpy.ops.mesh.primitive_cube_add(location=(cx + ancho, cy, altura_pared / 2))
```

- Los bloques **pares** usan `ParedOscura` con escala normal
- Los bloques **impares** usan `ParedDetalle` con escala Z de 1.5 (más altos), creando variación visual

  <img width="987" height="404" alt="Captura8" src="https://github.com/user-attachments/assets/fea89102-e9e3-495f-a15b-15db3f51ca82" />


---

### 5. Generación del Suelo (Malla Procedural)

En lugar de usar cubos, el suelo se genera como una **malla continua** de vértices y caras:

```python
for i in range(total_bloques):
    # Se calculan 2 vértices por bloque (izquierdo y derecho)
    verts.append((cx + px * (-ancho), cy + py * (-ancho), 0))
    verts.append((cx + px * ( ancho), cy + py * ( ancho), 0))

# Cada 4 vértices forman una cara cuadrangular
for i in range(total_bloques - 1):
    faces.append((a, b, c, d))
```

Esto garantiza que el suelo siga la curvatura del pasillo sin huecos ni intersecciones.


<img width="253" height="533" alt="Captura9" src="https://github.com/user-attachments/assets/539d9b90-e28f-435f-8b81-07cb9e827e77" />

---

### 6. Animación de Cámara (Mejora Principal)

Esta es la **mejora agregada** respecto a la tarea base. Se crea una cámara que recorre todo el pasillo mediante keyframes automáticos:

```python
# Configuración de la escena
fps = 24
duracion_s = 2
total_frames = fps * duracion_s
bpy.context.scene.frame_start = 1
bpy.context.scene.frame_end = total_frames

# Insertar keyframe en cada bloque
for i in range(total_bloques):
    frame = int(1 + (i / (bloques_kf - 1)) * (total_frames - 1))
    cam_obj.location = (cx, cy, cam_z)              # Posición al centro del pasillo
    cam_obj.rotation_euler = (math.radians(90), 0, rot)  # Apunta hacia adelante
    cam_obj.keyframe_insert(data_path="location", frame=frame)
    cam_obj.keyframe_insert(data_path="rotation_euler", frame=frame)
```

**Suavizado de interpolación:**
```python
for fcurve in action.fcurves:
    for kp in fcurve.keyframe_points:
        kp.interpolation = 'BEZIER'
```

Se usa interpolación **Bezier** en todos los keyframes para que la cámara no se detenga abruptamente entre bloques, logrando un movimiento cinematográfico fluido.

#### Parámetros de la cámara:
| Parámetro | Valor | Descripción |
|-----------|-------|-------------|
| `cam_z` | 1.6 | Altura de la cámara (nivel de ojos) |
| `lens` | 50mm | Lente de la cámara |
| `fps` | 24 | Fotogramas por segundo |
| `duracion_s` | 2 seg | Duración total del recorrido |
| Resolución | 1280×720 | HD 720p |

<img width="737" height="468" alt="Captura10" src="https://github.com/user-attachments/assets/c4f36c48-b8b1-4f3c-8135-274128f31da5" />


---

## Cómo ejecutar el proyecto

1. Abre **Blender** (versión 3.x o superior recomendada)
2. Ve a la pestaña **Scripting**
3. Crea un nuevo script y pega el contenido de `proyecto_pasillo.py`
4. Presiona **Run Script** (▶️)
5. El pasillo se generará automáticamente en la escena
6. Presiona **Espacio** en el viewport para reproducir la animación de la cámara
7. Para ver desde el punto de vista de la cámara: presiona **Numpad 0**

---

## Tecnologías utilizadas

- **Blender 3.x / 4.x** - Software de modelado y animación 3D
- **Python 3.x** - Lenguaje de scripting
- **bpy** - API de Python de Blender
- **math** - Librería estándar de Python para cálculos trigonométricos
