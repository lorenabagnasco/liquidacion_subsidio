# Sistema de Liquidación de Subsidios por Certificaciones Médicas

Aplicación desarrollada en **PHP + MySQL** que automatiza la liquidación del subsidio económico correspondiente a las certificaciones médicas de empleados.  
El sistema centraliza las certificaciones, evalúa múltiples condiciones laborales y genera los montos complementarios que deben pagarse para evitar pérdidas económicas durante los días certificados.

---

## 🚀 Funcionalidades principales

- Procesamiento automático de certificaciones médicas del mes.
- Evaluación de:
  - Cantidad de días certificados.
  - Continuación de certificaciones previas.
  - Internación o tratamientos especiales.
  - Retenciones de haberes (hijos, judiciales, etc.).
- Cálculo automático según:
  - Valor vigente de la BPC.
  - Montos cubiertos por BPS.
  - Haberes fijos, variables y promedios históricos.
- Generación de:
  - Recibos por empleado.
  - Archivos bancarios para depósito.
  - Resumen final por empresa.

---

## 🖥️ Capturas del sistema

> *(Aquí irán las 5 imágenes que vas a subir)*  

Te dejo el formato listo para completar:

### 📍 Formulario de certificación médica
![Formulario certificacion](img/creacion_certificacion_png)

### 📍 Certificacion generada y enviada al paciente
![Certificacion generada](img/certificacion_creada_y_enviada.png)

### 📍 Panel pre-liquidacion
![Panel pre-liquidacion](img/panel_antes_liquidar.png)

### 📍 Recibo generado para el empleado
![Recibo](img/recibo_subsidio.png)

### 📍 Listado de subsidios obtenidos
![Listado de Subsidios obtenidos](img/listado_subsidios.png)

---

## 🔧 Flujo general del proceso de liquidación

```mermaid
flowchart TD
A[Certificaciones creadas y cargadas en el sistema] --> B[Identificación de días y períodos]
B --> C[Evaluación: internación, continuidad, retenciones]
C --> D[Obtención de BPC y montos cubiertos por BPS]
D --> E[Cálculo del complemento]
E --> F[Generación de recibos]
E --> G[Generación de archivo bancario]
F --> H[Resumen final para la empresa]
