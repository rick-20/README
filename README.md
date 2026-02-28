# README
# ** Proyecto CONTROL VACUNACIÓN GANADERA **
El proyecto se realiza con el propósito de apoyar a las personas ganaderas a llevar un control de vacunación en sus ganados. Permitiendo dar de alta a sus animales y agregar las fechas requeridas en las cuales se
deberan vacunar a los mismos. Se pueden agregar a los dueños de los mismos, además de la comunidad a la que pertenecen. 
El proyecto resuelve la falta de administración que se lleva en el contexto de la misma administración de mantenimiento de los ganados.

# INSTALACIÓN
## APP.PY
### Este apartado es el "main" del proyecto, aqui se concentran los recursos como interfaces, base de datos, implementaciones e incluso APIs 
```python
from dotenv import load_dotenv
load_dotenv()

from flask import Flask, render_template, request, redirect, url_for, session
from flask_cors import CORS
from database.db import db

#Modelos
from models.ganadero import Ganadero
from models.animal import Animal

#Blueprints API
from routes.animal_routes import animal_bp
from routes.ganadero_routes import ganadero_bp
from routes.vacuna_routes import vacuna_bp
from routes.vacunacion_routes import vacunacion_bp
from routes.auth_routes import auth_bp

def create_app():
    app = Flask(__name__)
    app.config.from_object('config.Config')
    app.secret_key = "super_secret_key"
    CORS(app)
    db.init_app(app)
    # ==================================================
    #  LOGIN
    # ==================================================
    @app.route('/login', methods=['GET', 'POST'])
    def login():
        if request.method == 'POST':
            usuario = request.form['usuario']
            password = request.form['password']
            if usuario == "admin" and password == "1234":
                session['usuario'] = usuario
                return redirect(url_for('dashboard'))
            else:
                return "Credenciales incorrectas"
        return render_template('login.html')
    @app.route('/logout')
    def logout():
        session.clear()
        return redirect(url_for('login'))
    # ==================================================
    #  HOME
    # ==================================================
    @app.route('/')
    def home():
        if 'usuario' not in session:
            return redirect(url_for('login'))
        return render_template('index.html')
    # ==================================================
    #  DASHBOARD
    # ==================================================
    @app.route('/dashboard')
    def dashboard():
        if 'usuario' not in session:
            return redirect(url_for('login'))
        total_ganaderos = Ganadero.query.count()
        total_animales = Animal.query.count()
        return render_template(
            'dashboard.html',
            total_ganaderos=total_ganaderos,
            total_animales=total_animales
        )
    # ==================================================
    #  GANADEROS (CRUD)
    # ==================================================
    @app.route('/ganaderos')
    def vista_ganaderos():
        if 'usuario' not in session:
            return redirect(url_for('login'))
        ganaderos = Ganadero.query.all()
        return render_template('ganaderos.html', ganaderos=ganaderos)
    @app.route('/ganaderos/nuevo', methods=['GET', 'POST'])
    def crear_ganadero():
        if 'usuario' not in session:
            return redirect(url_for('login'))
        if request.method == 'POST':
            nuevo = Ganadero(
                nombre=request.form['nombre'],
                telefono=request.form['telefono']
            )
            db.session.add(nuevo)
            db.session.commit()
            return redirect(url_for('vista_ganaderos'))
        return render_template('crear_ganadero.html')
    @app.route('/ganaderos/eliminar/<int:id>')
    def eliminar_ganadero(id):
        if 'usuario' not in session:
            return redirect(url_for('login'))
        ganadero = Ganadero.query.get_or_404(id)
        db.session.delete(ganadero)
        db.session.commit()
        return redirect(url_for('vista_ganaderos'))
    # ==================================================
    #  ANIMALES (CRUD)   AHORA BIEN INDENTADO
    # ==================================================
    @app.route('/animales', methods=['GET', 'POST'])
    def vista_animales():
        if 'usuario' not in session:
            return redirect(url_for('login'))
        ganaderos = Ganadero.query.all()
        if request.method == 'POST':
            nuevo_animal = Animal(
                arete=request.form['arete'],
                nombre=request.form['nombre'],
                especie=request.form['especie'],
                edad=request.form['edad'],
                ganadero_id=request.form['ganadero_id']
            )
            db.session.add(nuevo_animal)
            db.session.commit()
            return redirect(url_for('vista_animales'))
        animales = Animal.query.all()
        total_animales = Animal.query.count()
        return render_template(
            'animales.html',
            animales=animales,
            ganaderos=ganaderos,
            total_animales=total_animales
        )
    # ==================================================
    #  REGISTRO DE BLUEPRINTS
    # ==================================================
    app.register_blueprint(auth_bp, url_prefix="/api/auth")
    app.register_blueprint(ganadero_bp, url_prefix="/api/ganaderos")
    app.register_blueprint(animal_bp, url_prefix="/api/animales")
    app.register_blueprint(vacuna_bp, url_prefix="/api/vacunas")
    app.register_blueprint(vacunacion_bp, url_prefix="/api/vacunaciones")
    # Crear tablas
    with app.app_context():
        db.create_all()
    return app
# Crear instancia
app = create_app()
if __name__ == "__main__":
    app.run(debug=True)
```
## config.py
```python
import os

class Config:
    SQLALCHEMY_DATABASE_URI = (
        f"mysql+pymysql://{os.getenv('DB_USER')}:"
        f"{os.getenv('DB_PASSWORD')}@"
        f"{os.getenv('DB_HOST')}:3306/"
        f"{os.getenv('DB_NAME')}"
    )
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    SECRET_KEY = os.getenv("SECRET_KEY")
```
# Carpeta models
## animal.py
```python
from database.db import db

class Animal(db.Model):
    __tablename__ = 'animales'

    id = db.Column(db.Integer, primary_key=True)
    arete = db.Column(db.String(50), unique=True, nullable=False)
    nombre = db.Column(db.String(100))
    especie = db.Column(db.String(50), nullable=False)
    edad = db.Column(db.Integer, nullable=False)

    # 🔹 Relación con Ganadero
    ganadero_id = db.Column(
        db.Integer,
        db.ForeignKey('ganaderos.id'),
        nullable=False
    )

    # 🔹 Un animal tiene muchas vacunaciones
    vacunaciones = db.relationship('Vacunacion', backref='animal', lazy=True)
```
## ganadero.py
```python
from database.db import db

class Ganadero(db.Model):
    __tablename__ = 'ganaderos'

    id = db.Column(db.Integer, primary_key=True)
    nombre = db.Column(db.String(100), nullable=False)
    telefono = db.Column(db.String(20))
    direccion = db.Column(db.String(200))

    # 🔹 Un ganadero tiene muchos animales
    animales = db.relationship('Animal', backref='ganadero', lazy=True)
```
## usuario.py
```python
from database.db import db

class Usuario(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    usuario = db.Column(db.String(50), unique=True, nullable=False)
    password = db.Column(db.String(100), nullable=False)
```
## vacuna.py
```python
from database.db import db

class Vacuna(db.Model):
    __tablename__ = 'vacunas'

    id = db.Column(db.Integer, primary_key=True)
    nombre = db.Column(db.String(100), nullable=False)
    enfermedad = db.Column(db.String(100), nullable=False)
    periodicidad_dias = db.Column(db.Integer, nullable=False)

    # 🔹 Una vacuna puede aplicarse muchas veces
    vacunaciones = db.relationship('Vacunacion', backref='vacuna', lazy=True)
```
## vacunacion.py
```python
from database.db import db
from datetime import date
from database.db import db

class Vacunacion(db.Model):
    __tablename__ = 'vacunaciones'

    id = db.Column(db.Integer, primary_key=True)
    fecha_aplicacion = db.Column(db.Date, nullable=False)
    observaciones = db.Column(db.String(255))

    # 🔹 Relación con Animal
    animal_id = db.Column(
        db.Integer,
        db.ForeignKey('animales.id'),
        nullable=False
    )

    # 🔹 Relación con Vacuna
    vacuna_id = db.Column(
        db.Integer,
        db.ForeignKey('vacunas.id'),
        nullable=False
    )
```
# Carpeta routes
## animal_routes.py
```python
from flask import Blueprint, jsonify
from models.animal import Animal

animal_bp = Blueprint('animal', __name__)

@animal_bp.route('/', methods=['GET'])
def listar_animales():
    animales = Animal.query.all()
    return jsonify([
        {
            "id": a.id,
            "arete": a.arete,
            "especie": a.especie,
            "edad": a.edad,
            "ganadero_id": a.ganadero_id
        }
        for a in animales
    ]), 200
```
## auth_routes.py
```python
from flask import Blueprint, request, jsonify
from models.usuario import Usuario
from database.db import db

auth_bp = Blueprint('auth', __name__)

@auth_bp.route('/usuarios', methods=['POST'])
def crear_usuario():
    data = request.json
    usuario = Usuario(
        usuario=data['usuario'],
        password=data['password']
    )
    db.session.add(usuario)
    db.session.commit()
    return jsonify({"mensaje": "Usuario creado"})
```
## ganadero_routes.py
```python
from flask import Blueprint, request, jsonify
from models.ganadero import Ganadero
from database.db import db

# Blueprint
ganadero_bp = Blueprint('ganadero', __name__)

#  Crear ganadero
@ganadero_bp.route('', methods=['POST'])
def crear_ganadero():
    data = request.get_json()

    if not data or not data.get('nombre') or not data.get('comunidad'):
        return jsonify({"error": "Faltan datos obligatorios"}), 400

    ganadero = Ganadero(
        nombre=data['nombre'],
        comunidad=data['comunidad']
    )

    db.session.add(ganadero)
    db.session.commit()

    return jsonify({
        "mensaje": "Ganadero registrado correctamente"
    }), 201


#  Listar ganaderos
@ganadero_bp.route('', methods=['GET'])
def listar_ganaderos():
    ganaderos = Ganadero.query.all()

    return jsonify([
        {
            "id": g.id,
            "nombre": g.nombre,
            "comunidad": g.comunidad
        }
        for g in ganaderos
    ]), 200
```
## vacuna_routes.py
```python
from flask import Blueprint, request, jsonify
from models.vacuna import Vacuna
from database.db import db

vacuna_bp = Blueprint('vacuna', __name__)

@vacuna_bp.route('/', methods=['POST'])
def crear_vacuna():
    data = request.json

    vacuna = Vacuna(
        nombre=data['nombre'],
        enfermedad=data['enfermedad'],
        periodicidad_dias=data['periodicidad_dias']
    )

    db.session.add(vacuna)
    db.session.commit()

    return jsonify({"mensaje": "Vacuna registrada correctamente"}), 201


@vacuna_bp.route('/', methods=['GET'])
def listar_vacunas():
    vacunas = Vacuna.query.all()
    return jsonify([
        {
            "id": v.id,
            "nombre": v.nombre,
            "enfermedad": v.enfermedad,
            "periodicidad_dias": v.periodicidad_dias
        }
        for v in vacunas
    ])
```
## vacunacion_routes.py
```python
from flask import Blueprint, jsonify
from datetime import date, timedelta
from models.vacunacion import Vacunacion
from models.animal import Animal
from database.db import db

# PRIMERO SE CREA EL BLUEPRINT
vacunacion_bp = Blueprint('vacunacion', __name__)

# ======================================================
# ALERTAS (VENCIDAS Y PRÓXIMAS)
# ======================================================

@vacunacion_bp.route('/alertas', methods=['GET'])
def alertas_vacunacion():
    hoy = date.today()
    proximos_dias = hoy + timedelta(days=7)

    vacunaciones = Vacunacion.query.all()

    vencidas = []
    proximas = []

    for v in vacunaciones:
        proxima_aplicacion = v.fecha_aplicacion + timedelta(days=v.vacuna.periodicidad_dias)

        info = {
            "animal_id": v.animal_id,
            "vacuna": v.vacuna.nombre,
            "enfermedad": v.vacuna.enfermedad,
            "proxima_aplicacion": proxima_aplicacion.strftime("%Y-%m-%d")
        }

        if proxima_aplicacion < hoy:
            vencidas.append(info)
        elif hoy <= proxima_aplicacion <= proximos_dias:
            proximas.append(info)

    return jsonify({
        "vencidas": vencidas,
        "proximas": proximas
    }), 200


# ======================================================
#  REPORTE POR GANADERO
# ======================================================

@vacunacion_bp.route('/reporte/ganadero/<int:ganadero_id>', methods=['GET'])
def reporte_por_ganadero(ganadero_id):

    animales = Animal.query.filter_by(ganadero_id=ganadero_id).all()

    if not animales:
        return jsonify({"mensaje": "Este ganadero no tiene animales registrados"}), 404

    reporte = []

    for animal in animales:
        for v in animal.vacunaciones:
            proxima_aplicacion = v.fecha_aplicacion + timedelta(days=v.vacuna.periodicidad_dias)

            reporte.append({
                "animal_id": animal.id,
                "animal_nombre": animal.nombre,
                "vacuna": v.vacuna.nombre,
                "enfermedad": v.vacuna.enfermedad,
                "fecha_aplicacion": v.fecha_aplicacion.strftime("%Y-%m-%d"),
                "proxima_aplicacion": proxima_aplicacion.strftime("%Y-%m-%d")
            })

    return jsonify(reporte), 200
```
# Carpeta templates
## alertas.html
```html
{% extends "base.html" %}

{% block content %}
<h2>Alertas de Vacunación</h2>

<h3>Vacunas Vencidas</h3>
<ul>
    {% for v in vencidas %}
        <li>{{ v.vacuna }} - {{ v.proxima_aplicacion }}</li>
    {% endfor %}
</ul>

<h3>Vacunas Próximas</h3>
<ul>
    {% for v in proximas %}
        <li>{{ v.vacuna }} - {{ v.proxima_aplicacion }}</li>
    {% endfor %}
</ul>
{% endblock %}
```
## animales.html
```html
{% extends "base.html" %}

{% block content %}

<h2 class="mb-3">🐂 Animales</h2>
<p>Total animales: <strong>{{ total_animales }}</strong></p>

<div class="card shadow mb-4">
    <div class="card-body">

        <form method="POST" class="row g-3">

            <div class="col-md-2">
                <input type="text" name="arete" class="form-control" placeholder="Arete" required>
            </div>

            <div class="col-md-2">
                <input type="text" name="nombre" class="form-control" placeholder="Nombre" required>
            </div>

            <div class="col-md-2">
                <input type="text" name="especie" class="form-control" placeholder="Especie" required>
            </div>

            <div class="col-md-2">
                <input type="number" name="edad" class="form-control" placeholder="Edad" required>
            </div>

            <div class="col-md-3">
                <select name="ganadero_id" class="form-select" required>
                    <option value="">Seleccionar ganadero</option>
                    {% for g in ganaderos %}
                        <option value="{{ g.id }}">{{ g.nombre }}</option>
                    {% endfor %}
                </select>
            </div>

            <div class="col-md-1">
                <button type="submit" class="btn btn-primary w-100">+</button>
            </div>

        </form>

    </div>
</div>

<div class="card shadow">
    <div class="card-body">

        <table class="table table-striped">
            <thead>
                <tr>
                    <th>Arete</th>
                    <th>Nombre</th>
                    <th>Especie</th>
                    <th>Edad</th>
                    <th>Ganadero</th>
                </tr>
            </thead>
            <tbody>
                {% for animal in animales %}
                <tr>
                    <td>{{ animal.arete }}</td>
                    <td>{{ animal.nombre }}</td>
                    <td>{{ animal.especie }}</td>
                    <td>{{ animal.edad }}</td>
                    <td>{{ animal.ganadero.nombre }}</td>
                </tr>
                {% endfor %}
            </tbody>
        </table>

    </div>
</div>

{% endblock %}
```
## base.html
```html
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Ganadera</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Bootstrap 5 -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>

<!-- NAVBAR -->
<nav class="navbar navbar-dark bg-dark px-3">
    
    <span class="navbar-brand mb-0 h1">🐄 Ganadera</span>

    <!-- BOTÓN HAMBURGUESA -->
    <div class="dropdown ms-auto">
        <button class="btn btn-outline-light"
                type="button"
                data-bs-toggle="dropdown"
                aria-expanded="false">
            ☰
        </button>

        <!-- MENÚ PEQUEÑO -->
        <ul class="dropdown-menu dropdown-menu-end shadow">

            <li>
                <a class="dropdown-item"
                   href="{{ url_for('dashboard') }}">
                    📊 Dashboard
                </a>
            </li>

            <li>
                <a class="dropdown-item"
                   href="{{ url_for('vista_ganaderos') }}">
                    👨‍🌾 Ganaderos
                </a>
            </li>

            <li>
                <a class="dropdown-item"
                   href="{{ url_for('vista_animales') }}">
                    🐂 Animales
                </a>
            </li>

            <li><hr class="dropdown-divider"></li>

            <li>
                <a class="dropdown-item text-danger"
                   href="{{ url_for('logout') }}">
                    🚪 Cerrar sesión
                </a>
            </li>

        </ul>
    </div>

</nav>


<!-- CONTENIDO DINÁMICO -->
<div class="container mt-4">
    {% block content %}
    {% endblock %}
</div>

<!-- Bootstrap JS -->
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/js/bootstrap.bundle.min.js"></script>

</body>
</html>
```
## crear_ganadero.html
```html
{% extends 'base.html' %}

{% block content %}
<h2>Nuevo Ganadero</h2>

<form method="POST">
    <input type="text" name="nombre" placeholder="Nombre" required>
    <input type="text" name="telefono" placeholder="Teléfono">
    <button type="submit">Guardar</button>
</form>

{% endblock %}
```
## dashboard.html
```html
{% extends "base.html" %}

{% block content %}

<h2 class="mb-4">📊 Panel de Control</h2>

<div class="row mb-4">
    <div class="col-md-6">
        <div class="card shadow text-white bg-success">
            <div class="card-body text-center">
                <h5>Total Ganaderos</h5>
                <h2>{{ total_ganaderos or 0 }}</h2>
            </div>
        </div>
    </div>

    <div class="col-md-6">
        <div class="card shadow text-white bg-info">
            <div class="card-body text-center">
                <h5>Total Animales</h5>
                <h2>{{ total_animales or 0 }}</h2>
            </div>
        </div>
    </div>
</div>

<div class="card shadow">
    <div class="card-body">
        <canvas id="grafica"></canvas>
    </div>
</div>

<!-- Chart.js -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<script>
document.addEventListener("DOMContentLoaded", function () {

    const datos = {
        ganaderos: {{ total_ganaderos | tojson }},
        animales: {{ total_animales | tojson }}
    };

    const ctx = document.getElementById('grafica');

    new Chart(ctx, {
        type: 'bar',
        data: {
            labels: ['Ganaderos', 'Animales'],
            datasets: [{
                label: 'Registros Totales',
                data: [datos.ganaderos, datos.animales]
            }]
        },
        options: {
            responsive: true
        }
    });

});
</script>


{% endblock %}
```
## ganaderos.html
```html
{% extends 'base.html' %}

{% block content %}

<h2>Ganaderos</h2>

<a href="/ganaderos/nuevo" class="btn btn-primary mb-3">+ Nuevo Ganadero</a>

<table class="table table-striped table-bordered">
    <thead class="table-dark">
        <tr>
            <th>Nombre</th>
            <th>Teléfono</th>
            <th>Acciones</th>
        </tr>
    </thead>
    <tbody>
    {% for g in ganaderos %}
        <tr>
            <td>{{ g.nombre }}</td>
            <td>{{ g.telefono }}</td>
            <td>
                <a href="/ganaderos/eliminar/{{ g.id }}" 
                   class="btn btn-danger btn-sm">
                   Eliminar
                </a>
            </td>
        </tr>
    {% endfor %}
    </tbody>
</table>

{% endblock %}
```
## index.html
```html
{% extends "base.html" %}

{% block content %}
<h2>Bienvenido al Sistema de Control de Vacunación Ganadera </h2>

<p>Administra ganaderos, animales y vacunaciones desde este panel.</p>
{% endblock %}
```
## login.html
```html
<!DOCTYPE html>
<html>
<head>
    <title>Login</title>
    <meta charset="UTF-8">

    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.2/dist/css/bootstrap.min.css" rel="stylesheet">

    <style>
        body {
            margin: 0;
            height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;

            background: linear-gradient(rgba(0,0,0,0.5), rgba(0,0,0,0.5)),
                        url('https://estamosenlaweb.com.ar/upload/products/4722/fondo.jpg');

            background-size: cover;
            background-position: center;
            background-repeat: no-repeat;
        }

        .card {
            width: 380px;
            border-radius: 15px;
        }
    </style>
</head>

<body>

<div class="card p-4 shadow">
    <h3 class="text-center mb-3 text-primary"> Iniciar Sesión</h3>

    <form method="POST">
        <input type="text" name="usuario" class="form-control mb-3" placeholder="Usuario" required>
        <input type="password" name="password" class="form-control mb-3" placeholder="Contraseña" required>
        <button class="btn btn-primary w-100">Ingresar</button>
    </form>
</div>

</body>
</html>
```
# .env
```
DB_USER=root
DB_PASSWORD=1234
DB_HOST=localhost
DB_NAME=vacunacion_ganadera
SECRET_KEY=1234
```
