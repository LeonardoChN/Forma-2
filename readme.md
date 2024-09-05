# Forma N°2 
----
### Instalar python y Verificar que venga con PIP (Al momento de instalar)
[Descarga Python](https://www.python.org/downloads/)
----
- Instalar Django en SH
```sh
pip install django
```

- Iniciar django en SH
```sh
django-admin startproject Reconocimiento_Facial

cd Reconocimiento_Facial

python manage.py startapp usuarios
```
- Instalacion Necesarias a travez de SH
```sh
pip install django opencv-python-headless tensorflow Pillow
```
- ## EN VSCode
- #### Configuracion Settings.py
`se agrega: `
`"usuarios" a INSTALLED_APPS. (reconocimiento_Facial/settings.py)`
```python
INSTALLED_APPS = [
    'usuarios',
]
```

- Modelo de Datos
```python 
# usuarios/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class Usuario(AbstractUser):
    rut = models.CharField(max_length=12, unique=True)
    embedding_facial = models.BinaryField()  # Aquí almacenaremos el embedding del rostro
```

- #### Crear Carpeta para los templates
En la carpeta del proyecto (`"Reconocimiento_facial"`), creamos una carpeta llamada `"templates" `
luego otra carpeta llamada igual que la aplicacion (`"usuarios"`)
y dentro de esa carpeta agregamos los .html

----

- #### Agregar los Templates a settings.py `(reconocimiento_Facial/settings.py)`

En el settings.py importamos OS
```python
import os
```
Y agregamos la carpeta en la parte de `'DIRS'` tal cual esta en el codigo 
(solo esa parte se modifica)

```python
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(BASE_DIR, 'templates')] ##Esta se modifica,
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                ...
            ],
        },
    },
]
```
----
### Templates
----

- Template Base (Crear el .html)
```html
<!-- templates/usuarios/base.html -->
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block title %}Reconocimiento Facial{% endblock %}</title>
    <!-- Bootstrap CSS -->
    <link href="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/css/bootstrap.min.css" rel="stylesheet">
    <!-- Custom CSS -->
    <link rel="stylesheet" href="{% static 'css/styles.css' %}">
    {% block extra_css %}{% endblock %}
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
        <a class="navbar-brand" href="{% url 'home' %}">Reconocimiento Facial</a>
    </nav>
    <div class="container mt-5">
        {% block content %}{% endblock %}
    </div>
    <!-- Bootstrap JS and dependencies -->
    <script src="https://code.jquery.com/jquery-3.5.1.slim.min.js"></script>
    <script src="https://cdn.jsdelivr.net/npm/@popperjs/core@2.11.6/dist/umd/popper.min.js"></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/js/bootstrap.min.js"></script>
    {% block extra_js %}{% endblock %}
</body>
</html>
```

- Templates registro
```html
<!-- templates/usuarios/registro.html -->
{% extends 'base.html' %}

{% block title %}Registro de Usuario{% endblock %}

{% block content %}
<div class="card mx-auto" style="max-width: 500px;">
    <div class="card-header bg-primary text-white text-center">
        <h4>Registro de Usuario</h4>
    </div>
    <div class="card-body">
        <form method="post">
            {% csrf_token %}
            <div class="form-group">
                <label for="nombre">Nombre Completo</label>
                <input type="text" class="form-control" id="nombre" name="nombre" required>
            </div>
            <div class="form-group">
                <label for="rut">RUT</label>
                <input type="text" class="form-control" id="rut" name="rut" required>
            </div>
            <div class="form-group text-center">
                <button type="button" class="btn btn-secondary" id="iniciar-captura">Iniciar Captura</button>
            </div>
            <div class="form-group text-center">
                <video id="video" class="border" style="width: 100%; max-width: 400px;" autoplay></video>
            </div>
            <div class="form-group text-center">
                <button type="submit" class="btn btn-primary">Registrar</button>
            </div>
        </form>
    </div>
</div>
{% endblock %}

{% block extra_js %}
<script>
    // JavaScript para manejar la cámara y mostrar la vista previa del video
    const video = document.getElementById('video');
    const btnCaptura = document.getElementById('iniciar-captura');

    btnCaptura.addEventListener('click', () => {
        navigator.mediaDevices.getUserMedia({ video: true }).then((stream) => {
            video.srcObject = stream;
        });
    });
</script>
{% endblock %}

```


- Template Login
```html
<!-- templates/usuarios/login.html -->
{% extends 'base.html' %}

{% block title %}Inicio de Sesión{% endblock %}

{% block content %}
<div class="card mx-auto" style="max-width: 500px;">
    <div class="card-header bg-success text-white text-center">
        <h4>Iniciar Sesión</h4>
    </div>
    <div class="card-body">
        <form method="post">
            {% csrf_token %}
            <div class="form-group">
                <label for="rut">RUT</label>
                <input type="text" class="form-control" id="rut" name="rut" required>
            </div>
            <div class="form-group text-center">
                <button type="button" class="btn btn-secondary" id="iniciar-login">Iniciar Login</button>
            </div>
            <div class="form-group text-center">
                <video id="video" class="border" style="width: 100%; max-width: 400px;" autoplay></video>
            </div>
            <div class="form-group text-center">
                <button type="submit" class="btn btn-success">Iniciar Sesión</button>
            </div>
            {% if error %}
                <div class="alert alert-danger text-center">
                    {{ error }}
                </div>
            {% endif %}
        </form>
    </div>
</div>
{% endblock %}

{% block extra_js %}
<script>
    // JavaScript para manejar la cámara y mostrar la vista previa del video
    const video = document.getElementById('video');
    const btnLogin = document.getElementById('iniciar-login');

    btnLogin.addEventListener('click', () => {
        navigator.mediaDevices.getUserMedia({ video: true }).then((stream) => {
            video.srcObject = stream;
        });
    });
</script>
{% endblock %}

```

----
### Vistas
----

- Vista Registro
```python
# usuarios/views.py
from django.shortcuts import render, redirect
from .models import Usuario
from django.contrib.auth import login
import cv2
import numpy as np
from tensorflow.keras.models import load_model

threshold:float = 0.6 #Este servira para el LOGIN

def registrar(request):
    if request.method == 'POST':
        nombre = request.POST['nombre']
        rut = request.POST['rut']
        # Capturar el video usando OpenCV
        cap = cv2.VideoCapture(0)
        face_embeddings = []
        model = load_model('path_to_facenet_model.h5')  # Cargar el modelo FaceNet

        while len(face_embeddings) < 10:  # Captura 10 frames para tener una buena muestra
            ret, frame = cap.read()
            # Procesar cada frame para extraer el embedding facial
            # (Implementar la lógica para detectar el rostro y obtener embeddings)
            embedding = model.predict(frame)  # Suponiendo que el modelo ya está preparado
            face_embeddings.append(embedding)
        
        cap.release()
        avg_embedding = np.mean(face_embeddings, axis=0)  # Promediar los embeddings
        # Guardar el usuario en la base de datos
        usuario = Usuario.objects.create_user(username=rut, rut=rut, embedding_facial=avg_embedding.tobytes())
        usuario.save()
        login(request, usuario)
        return redirect('menu_principal')

    return render(request, 'registro.html')
```

-  Vista Login
```python
# usuarios/views.py
def login_view(request):
    if request.method == 'POST':
        rut = request.POST['rut']
        # Capturar la imagen del rostro
        cap = cv2.VideoCapture(0)
        ret, frame = cap.read()
        cap.release()
        # Obtener embedding del rostro
        model = load_model('path_to_facenet_model.h5')
        embedding = model.predict(frame)
        # Buscar usuario en la base de datos
        try:
            usuario = Usuario.objects.get(rut=rut)
            # Comparar embeddings
            stored_embedding = np.frombuffer(usuario.embedding_facial, dtype=np.float32)
            if np.linalg.norm(stored_embedding - embedding) < threshold:  # Definir un umbral
                login(request, usuario)
                return redirect('menu_principal')
            else:
                return render(request, 'login.html', {'error': 'Desconocido'})
        except Usuario.DoesNotExist:
            return render(request, 'login.html', {'error': 'Usuario no encontrado'})

    return render(request, 'login.html')
```



- #### Configurar los URLS.PY
`FALTA TERMINAR DE CONFIGURAR LOS URLS TANTO EN EL PROYECTO COMO EN LA APP`

## DE ACA PARA ABAJO ESTA EN VEREMOS xD
- ## Migrar el modelo a sqlite
```sh
python manage.py makemigrations
```

```sh
python manage.py migrate
```

