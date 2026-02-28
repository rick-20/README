# README
# ** Proyecto CONTROL VACUNACIÓN GANADERA **
El proyecto se realiza con el propósito de apoyar a las personas ganaderas a llevar un control de vacunación en sus ganados. Permitiendo dar de alta a sus animales y agregar las fechas requeridas en las cuales se
deberan vacunar a los mismos. Se pueden agregar a los dueños de los mismos, además de la comunidad a la que pertenecen. 
El proyecto resuelve la falta de administración que se lleva en el contexto de la misma administración de mantenimiento de los ganados.

# INSTALACIÓN
## APP.PY
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
