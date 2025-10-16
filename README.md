> Si estás en la sede y tu Python es 3.8 ejecuta el siguiente comando:
```bash
C:\ProgramData\anaconda3\Scripts\activate.bat
```

Una vez estés en el repo con una versión de Python actualizada crea un VENV

```bash
python -m venv venv
```

Activa el VENV

```bash
venv\Scripts\activate
```

Crea un archivo `requirements.txt` para las dependencias de la aplicación

```txt
django
python-dotenv
gunicorn
psycopg2
```

* Django: Framework backend a utilizar
* Python .env: Leer variables de entorno
* Gunicorn: Utilizar para WSGI
* PsycoPG2: Conector para PostgreSQL

Una vez activado el ambiente deben de instalar las dependencias

```bash
pip install -r requirements.txt
```

Creamos el proyecto "main" en la raíz del repo

```bash
django-admin startproject main .
```

Creamos la app venta

```bash
django-admin startapp venta
```

Instalan la aplicación en `main/settings.py`

```py
...

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'venta'
]
```

Configuramos la conexión a PostgreSQL en `main/settings.py`

```py
...

from dotenv import load_dotenv
import os
load_dotenv()

...

DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.getenv("dbname"),
        "USER": os.getenv("user"),
        "PASSWORD": os.getenv("password"),
        "HOST": os.getenv("host"),
        "PORT": os.getenv("port"),
    }
}

```


Ahora, deben de crear las variables de entorno con un archivo `.env`.
Las credenciales se encuentran en el boton *Connect* en el apartado *Transaction pooler*

```env
user=USER
password=PASSWORD
host=HOST
port=PORT
dbname=NAME
```

Crean modelos

```py
from django.db import models


class Producto(models.Model):
    nombre = models.CharField(max_length=120, db_index=True)
    sku = models.CharField(max_length=50, unique=True)
    precio = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.PositiveIntegerField(default=0)
    activo = models.BooleanField(default=True)

    def __str__(self):
        return f"{self.nombre} ({self.sku})"


class Cliente(models.Model):
    nombre = models.CharField(max_length=120)
    email = models.EmailField(blank=True, null=True)

    def __str__(self):
        return self.nombre


class Venta(models.Model):
    cliente = models.ForeignKey(Cliente, on_delete=models.PROTECT)
    fecha = models.DateTimeField(auto_now_add=True)
    anulada = models.BooleanField(default=False)

    @property
    def total(self):
        return sum(d.subtotal for d in self.detalles.all())

    def __str__(self):
        return f"Venta #{self.pk} — {self.cliente}"


class DetalleVenta(models.Model):
    venta = models.ForeignKey(
        Venta, related_name='detalles', on_delete=models.CASCADE)
    producto = models.ForeignKey(Producto, on_delete=models.PROTECT)
    cantidad = models.PositiveIntegerField()
    precio_unitario = models.DecimalField(max_digits=10, decimal_places=2)

    @property
    def subtotal(self):
        return self.cantidad * self.precio_unitario

    def __str__(self):
        return f"{self.producto} x{self.cantidad}"

```

Registran en admin

```py
from django.contrib import admin
from .models import Producto, Cliente, Venta, DetalleVenta

@admin.register(Producto)
class ProductoAdmin(admin.ModelAdmin):
    list_display = ("nombre", "sku", "precio", "stock", "activo")
    search_fields = ("nombre", "sku")  # texto rápido
    list_filter = ("activo",)  # filtros laterales
    ordering = ("nombre",)
    list_per_page = 25
    autocomplete_fields = ()  # ej.: ("categoria",) si existiera FK grande

@admin.register(Cliente)
class ClienteAdmin(admin.ModelAdmin):
    list_display = ("nombre", "email")
    search_fields = ("nombre", "email")

class DetalleVentaInline(admin.TabularInline):
    model = DetalleVenta
    extra = 1
    autocomplete_fields = ("producto",)
    readonly_fields = ("subtotal",)

@admin.register(Venta)
class VentaAdmin(admin.ModelAdmin):
    date_hierarchy = "fecha"
    list_display = ("id", "cliente", "fecha", "anulada", "total_display")
    search_fields = ("cliente__nombre", "id")  # lookups a relacionadas
    list_filter = ("anulada",)
    inlines = [DetalleVentaInline]

    @admin.display(description="Total", ordering="id")
    def total_display(self, obj):
        return f"${obj.total:,.0f}"

```


Crear un superuser (admin, admin) con bypass

```bash
python manage.py createsuperuser
```


Hacer migraciones
```bash
python manage.py makemigrations
python manage.py migrate
```

Ejecutan la app e [Inician sesión](http://127.0.0.1:8000/admin/login/?next=/admin/) 
````bash
python manage.py runserver
```