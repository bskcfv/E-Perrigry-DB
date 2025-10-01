# E-Perrigry-DB

# Sistema de Ventas – Base de Datos en PostgreSQL

Este proyecto define la estructura de una base de datos para un **sistema de ventas con carrito de compras**. Incluye tablas, secuencias, procedimientos almacenados, triggers, funciones y vistas para manejar clientes, administradores, productos, carritos y ventas.

---

## 📂 Estructura de la BD

### Tablas principales

* **Administrador** → Gestión de administradores.
* **Cliente** → Información de clientes registrados.
* **Producto** → Catálogo de productos disponibles.
* **Carrito_compra** → Carritos activos por cliente.
* **Detalle_carrito** → Relación entre carritos y productos.
* **Venta** → Compras confirmadas.
* **Detalle_venta** → Relación entre ventas y productos vendidos.

Cada tabla usa secuencias (`_seq`) para IDs autoincrementales.

---

## ⚙️ Procedimientos almacenados (Stored Procedures)

* `anhadir_carrito(cliente, producto, cantidad)` → Añade productos al carrito validando stock.
* `registrar_venta(cliente)` → Registra una venta para un cliente tomando como base los productos de su carrito, pero permite ajustar la cantidad comprada (incluso mayor a la que estaba en el carrito), siempre y cuando haya stock disponible.
* `p_insert_productos(nombre, precio, stock)` → Inserta nuevos productos.
* `p_update_producto(id, precio, stock)` → Actualiza productos existentes.
* `p_delete_producto(id)` → Elimina productos.

---

## 🧩 Funciones

* **Login**

  * `f_login_admin(email)`
  * `f_login_cliente(email)`
* **Registro**

  * `f_registar_cliente(nombre, email, password)` → Registra un cliente nuevo validando correo único.

---

## 🔄 Triggers

* `f_actualizar_stock` → Reduce stock tras una venta.
* `f_update_carrito` → Actualiza el total de un carrito al insertar/actualizar detalle.
* `f_after_update_detalle_carrito` → Recalcular el Precio Total del Carrito en Base a un nuevo ingreso o Actualización en 'Detalles de Carrito'.
* `f_delete_detalle_carrito` → Borra el registro de 'detalles del carrito' al estar vacío tras una compra realizada.
* `f_register_carrito` → Crea carrito automáticamente al registrar cliente.

---

## 📊 Vistas (Reports & Analytics)

* `info_cliente` → Información básica de clientes.
* `promedio_cliente` → Promedio de gasto por cliente.
* `tendencias_by_doy` → Tendencias de ventas por día del año.
* `tendencias_by_month` → Ventas agrupadas por mes.
* `tendencias_productos` → Productos más vendidos.
* `top_productos` → Ranking de productos top.

---


