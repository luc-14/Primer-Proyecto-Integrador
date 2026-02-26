# Primer-Proyecto-Integrador
Este proyecto utiliza la API de Python para Blender (bpy) para generar un entorno tridimensional de forma procedural. El escenario consiste en un pasillo con tramos rectos y curvos, donde la geometría y los materiales se asignan dinámicamente mediante algoritmos.

La característica principal de esta entrega es la automatización de la cámara, la cual recorre el pasillo siguiendo una trayectoria matemática calculada (interpolación circular) para ofrecer un recorrido fluido del escenario generado.
# Funcionalidades Principales
1. Generación de Geometría Procedural

Tramo Recto: Generación de muros laterales con alternancia de materiales basada en índices (módulo).

Tramo Curvo: Cálculo de posiciones mediante funciones trigonométricas ($sin$ y $cos$) para crear una curva perfecta de 90 grados.

Materiales Dinámicos: Creación de materiales mediante nodos (Principled BSDF) configurados totalmente por código.

2. Sistema de Animación por Keyframes

El script no solo crea los objetos, sino que inserta fotogramas clave (keyframes) automáticamente:

Frames 1-100: Recorrido lineal por el pasillo principal.

Frames 101-250: Transición curva calculada paso a paso para mantener la cámara centrada en el camino y rotarla suavemente hacia la dirección del movimiento.

# Cómo Ejecutar el Script
1. Abre Blender
<img width="1919" height="1138" alt="image" src="https://github.com/user-attachments/assets/12e33b71-b31d-4b4e-9b0b-210902087d14" />

2. Ve a la pestaña Scripting en la parte superior
<img width="1918" height="1137" alt="image" src="https://github.com/user-attachments/assets/55eab8c5-bf8b-493f-9455-50e8286efb8b" />

3. Haz clic en New y pega el codigo proporcionado posteriormente
<img width="1918" height="1141" alt="image" src="https://github.com/user-attachments/assets/10f56549-8345-4819-b9f3-79789dce1eaf" />

4. Codigo
```python
import bpy
import random
import math 

def crear_material(nombre, color_rgb):
    mat = bpy.data.materials.new(name=nombre)
    mat.use_nodes = True
    bsdf = mat.node_tree.nodes["Principled BSDF"]
    bsdf.inputs['Base Color'].default_value = (*color_rgb, 1.0) 
    bsdf.inputs['Roughness'].default_value = 0.7 
    return mat

def generar_escenario():
    # 1. Limpiar la escena previa
    bpy.ops.object.select_all(action='SELECT')
    bpy.ops.object.delete()

    # 2. Definir materiales
    mat_pared_a = crear_material("ParedOscura", (0.1, 0.1, 0.1))
    mat_pared_b = crear_material("ParedDetalle", (0.8, 0.2, 0.0))

    # 3. Parámetros del escenario
    largo_pasillo = 10
    ancho_pasillo = 4
    radio_curva = 12 

    # 4a. Tramo Recto
    for i in range(largo_pasillo):
        bpy.ops.mesh.primitive_cube_add(location=(-ancho_pasillo, i * 2, 1))
        pared_izq = bpy.context.active_object
        if i % 2 == 0:
            pared_izq.data.materials.append(mat_pared_a)
        else:
            pared_izq.data.materials.append(mat_pared_b)
            pared_izq.scale.z = 1.5

        bpy.ops.mesh.primitive_cube_add(location=(ancho_pasillo, i * 2, 1))
        pared_der = bpy.context.active_object
        pared_der.data.materials.append(mat_pared_a)

    # 4b. Tramo Curvo
    cy = (largo_pasillo - 1) * 2 
    cx = radio_curva 

    for j in range(1, largo_pasillo + 1):
        angulo = math.pi - (j * (math.pi / 2) / largo_pasillo)
        rotacion_z = math.pi - angulo 

        # Pared Izquierda Curva
        x_izq = cx + (radio_curva + ancho_pasillo) * math.cos(angulo)
        y_izq = cy + (radio_curva + ancho_pasillo) * math.sin(angulo)
        bpy.ops.mesh.primitive_cube_add(location=(x_izq, y_izq, 1), rotation=(0, 0, rotacion_z))
        pared_izq_curva = bpy.context.active_object
        
        if (i + j) % 2 == 0:
            pared_izq_curva.data.materials.append(mat_pared_a)
        else:
            pared_izq_curva.data.materials.append(mat_pared_b)
            pared_izq_curva.scale.z = 1.5

        # Pared Derecha Curva
        x_der = cx + (radio_curva - ancho_pasillo) * math.cos(angulo)
        y_der = cy + (radio_curva - ancho_pasillo) * math.sin(angulo)
        bpy.ops.mesh.primitive_cube_add(location=(x_der, y_der, 1), rotation=(0, 0, rotacion_z))
        pared_der_curva = bpy.context.active_object
        pared_der_curva.data.materials.append(mat_pared_a)

    # 5. Agregar suelo
    bpy.ops.mesh.primitive_plane_add(size=1, location=(radio_curva/2, cy/2 + radio_curva/2, 0))
    suelo = bpy.context.active_object
    suelo.scale.x = (ancho_pasillo * 2) + radio_curva + 10
    suelo.scale.y = (largo_pasillo * 2) + radio_curva + 10

    # 6. Agregar Luces
    bpy.ops.object.light_add(type='SUN', location=(10, 10, 10), rotation=(math.radians(-45), math.radians(30), 0))
    sun = bpy.context.active_object
    sun.data.energy = 3 

    bpy.ops.object.light_add(type='POINT', location=(0, 5, 4))
    luz1 = bpy.context.active_object
    luz1.data.energy = 500 

    bpy.ops.object.light_add(type='POINT', location=(radio_curva, cy + radio_curva/2, 4))
    luz2 = bpy.context.active_object
    luz2.data.energy = 800

    # 7. Agregar Cámara (Ajustamos altura a Z=1.5 para simular vista humana)
    bpy.ops.object.camera_add(location=(0, -6, 1.5), rotation=(math.radians(90), 0, 0))
    camara = bpy.context.active_object
    bpy.context.scene.camera = camara
    camara.rotation_mode = 'XYZ' # Necesario para animar la rotación correctamente

    # --- 8. ANIMACIÓN DEL RECORRIDO ---
    
    # Configuramos la duración de la animación a 250 fotogramas (~10 segundos)
    bpy.context.scene.frame_start = 1
    bpy.context.scene.frame_end = 250

    # Fotograma 1: Inicio del pasillo
    camara.location = (0, -6, 1.5)
    camara.rotation_euler = (math.radians(90), 0, 0) # Mirando al frente (Norte)
    camara.keyframe_insert(data_path="location", frame=1)
    camara.keyframe_insert(data_path="rotation_euler", frame=1)

    # Fotograma 100: Final del tramo recto
    camara.location = (0, cy, 1.5)
    camara.rotation_euler = (math.radians(90), 0, 0)
    camara.keyframe_insert(data_path="location", frame=100)
    camara.keyframe_insert(data_path="rotation_euler", frame=100)

    # Fotogramas 101 a 250: Tramo Curvo (Animamos cuadro por cuadro para una curva perfecta)
    for frame in range(101, 251):
        progreso = (frame - 100) / 150.0 # Va de 0.0 a 1.0 a lo largo de 150 frames
        angulo_curva = math.pi - (progreso * (math.pi / 2)) # De 180° a 90°
        
        # Calculamos la posición central en la curva
        x_cam = cx + radio_curva * math.cos(angulo_curva)
        y_cam = cy + radio_curva * math.sin(angulo_curva)
        
        # Rotamos la cámara hacia la derecha a medida que avanza
        rot_z = -progreso * (math.pi / 2) 
        
        camara.location = (x_cam, y_cam, 1.5)
        camara.rotation_euler = (math.radians(90), 0, rot_z)
        
        camara.keyframe_insert(data_path="location", frame=frame)
        camara.keyframe_insert(data_path="rotation_euler", frame=frame)

generar_escenario()
```
5. Presiona el botón Run Script.
<img width="1918" height="1142" alt="image" src="https://github.com/user-attachments/assets/da72c9ba-dee4-45b1-a60d-b231b1b0cefd" />

# Resultado

https://github.com/user-attachments/assets/fd49f2a6-f8bd-45a7-9cd2-47a10ec73666


