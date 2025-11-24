# Sistema de Liquidación de Subsidios por Certificaciones Médicas

Aplicación desarrollada en **PHP + MySQL** que automatiza la liquidación del subsidio económico correspondiente a las certificaciones médicas de empleados.  
El sistema centraliza las certificaciones, evalúa múltiples condiciones laborales y genera los montos complementarios que deben pagarse para evitar pérdidas económicas durante los días certificados.

---

## 🚀 Funcionalidades principales

- Procesamiento automático de certificaciones médicas del mes.
- Carga de Archivo brindado por la Empresa con los haberes fijos y variables de los ultimos 6 meses.
- Procesamiento y calculo de haberes por funcionario certificado.
- Evaluación de:
  - Cantidad de días certificados.
  - Continuación de certificaciones previas.
  - Internación o tratamientos especiales.
  - Retenciones de haberes (hijos, judiciales, etc.).
- Cálculo automático según:
  - Valor vigente de la BPC.
  - Montos cubiertos por BPS.
- Generación de:
  - Recibos por empleado.
  - Archivos bancarios para depósito.
  - Resumen final por empresa.

---

## 🖥️ Capturas del sistema


### 📍 Formulario de creació de certificación médica
![Formulario certificacion](creacion_certificacion.png)

### 📍 Certificacion generada y enviada al paciente
![Certificacion generada](certificacion_creada_y_enviada.png)

### 📍 Panel de liquidacion
![Panel pre-liquidacion](panel_antes_liquidar.png)



### 📍 Listado de subsidios obtenidos
![Listado de Subsidios obtenidos](listado_subsidios.png)

---

## 🔧 Flujo general del proceso de liquidación
```mermaid

flowchart TD
A[Certificaciones creadas y cargadas en el sistema] --> B[Carga de Archivo con Haberes Fijos y Variables]
B --> C[Identificación de días, períodos, internación, continuidad, retenciones]
C --> D[Obtención de BPC y montos cubiertos por BPS]
D --> E[Cálculo del complemento]
E --> F[Generación de recibos]
E --> G[Generación de archivos bancarios]
E --> H[Resumen final para la empresa]
```


🧩 Cálculo del complemento según días cubiertos por BPS y la empresa

Esta sección del sistema calcula cuánto corresponde pagar por cada certificación, considerando los días cubiertos por BPS, los que cubre la empresa y los días no pagos según la normativa.

La empresa comienza a pagar a partir del tercer día certificado, por lo que los primeros dos días del período se restan automáticamente.
Esta información ya está parametrizada y proviene de base de datos (campo dias_menos).

📌 ¿Qué calcula este módulo?

Valor diario de salario según BPS.

Valor diario de salario según empresa.

Días abonados por BPS.

Días abonados por la empresa.

Días no cubiertos (los primeros dos días).

Diferencia que debe pagar la empresa luego de descontar lo que cubre BPS.

Proporcional de aguinaldo.

📌  Cálculo del complemento SEFMU y monto cubierto por BPS

```php
$por_dia_bps = round(($sueldo_base['sueldo_bps'] / 30), 2);
$liquidacion_bps = $por_dia_bps * $certPer['dias_bps'];

$liquidacion_sefmu = 0;

$por_dia = ($sueldo_base['sueldo_sefmu'] / 30);
$liquidacion_sefmu = $por_dia * ($certPer['periodo_cant_dias'] - $certPer['dias_menos']);

$nominal = $liquidacion_sefmu;
$liquido_a_pagar_sefmu = $liquidacion_sefmu - $liquidacion_bps;

// Calcular el aguinaldo
$liquido_sefmu_sin_a = $liquido_a_pagar_sefmu;
$aguinaldo = $liquido_sefmu_sin_a / 12;

$liquido_a_pagar_sefmu = $liquido_sefmu_sin_a + $aguinaldo;
```

📌 Resumen del cálculo

✔ Se identifica cuántos días paga BPS y cuántos paga la empresa.

✔ Se descartan automáticamente los días no cubiertos (primeros 2).

✔ Se calcula el complemento económico.

✔ Se agrega concepto de aguinaldo proporcional.

✔ Resultado final listo para recibo y archivo bancario.


### 🧩 Detección de certificaciones continuadas

En este paso, el sistema verifica si la certificación actual **continúa inmediatamente** de la certificación anterior del mismo empleado.  
Esto es importante porque afecta el cálculo total de días certificados y también determina la cantidad de días **no cubiertos** por la empresa (que comienza a pagar recién a partir del día 3, valor obtenido desde una tabla configurada).

```php
$fin_licencia_anterior = $cert_de_funcionario[$i-1]['fin_licencia'];
$incio_licencia_actual = $certificacion['inicio_licencia'];

// El inicio esperado es el día siguiente al fin de la licencia anterior
$inicio_que_debe_ser = date("Y-m-d", strtotime($fin_licencia_anterior . "+ 1 days"));

if ($incio_licencia_actual == $inicio_que_debe_ser) {
    /**
     * Si el fin de la licencia anterior es exactamente
     * el día previo al inicio de la licencia actual,
     * entonces la certificación se considera continuada.
     */
    $continua_licencia = 1;
} else {
    /**
     * Si no, se trata de una certificación independiente.
     */
    $continua_licencia = 0;
}
```

#### ✔ ¿Qué resuelve este bloque?
- Detecta si la certificación **es continuación** de otra previa.  
- Permite unir correctamente los días para el cálculo del subsidio.  
- Determina cuántos días deben descontarse según la regla interna:  
  **la empresa comienza a cubrir a partir del día 3**,  
  obtenido desde la tabla de parámetros (`dias_menos`).  
- Asegura que no se liquiden días de más o de menos en casos de certificaciones encadenadas.

---
### 🧮 Distribución de días cubiertos por BPS entre períodos del mes

En este sistema, las certificaciones médicas pueden abarcar dos períodos distintos dentro del mes (por ejemplo, fin de mes → comienzo del mes siguiente).  
Por eso es necesario determinar cuántos días cubre BPS en cada uno de esos períodos:

- **Primera inserción:** días del mes inicial  
- **Segunda inserción:** días del mes final  

El algoritmo compara:

- `$cant_dias` → días totales certificados  
- `$dias_bps` → días cubiertos por BPS  
- `$can_numero` → días del primer mes  
- `$can_numero_mes_final` → días del segundo mes  

Según la diferencia entre los días totales y los días cubiertos por BPS, se aplican reglas para repartir los días correctamente.

```php
if ($cant_dias == $dias_bps) {
    // Si BPS cubre todos los días, los días se distribuyen igual en ambos períodos
    $dias_bps_primera_insersion = $can_numero;
    $dias_bps_segunda_insersion = $can_numero_mes_final;

} else {

    // Diferencia de 3 días entre total y días BPS
    if ($cant_dias - $dias_bps == 3) {
        if ($can_numero > 3) {
            $dias_bps_primera_insersion = $can_numero - 3;
            $dias_bps_segunda_insersion = $can_numero_mes_final;

        } else if ($can_numero == 3) {
            $dias_bps_primera_insersion = 0;
            $dias_bps_segunda_insersion = $can_numero_mes_final;

        } else if ($can_numero == 2) {
            $dias_bps_primera_insersion = 0;
            $dias_bps_segunda_insersion = $can_numero_mes_final - 1;

        } else if ($can_numero == 1) {
            $dias_bps_primera_insersion = 0;
            $dias_bps_segunda_insersion = $can_numero_mes_final - 2;
        }

    // Diferencia de 2 días
    } else if ($cant_dias - $dias_bps == 2) {
        if ($can_numero > 2) {
            $dias_bps_primera_insersion = $can_numero - 2;
            $dias_bps_segunda_insersion = $can_numero_mes_final;

        } else if ($can_numero == 2) {
            $dias_bps_primera_insersion = 0;
            $dias_bps_segunda_insersion = $can_numero_mes_final;

        } else if ($can_numero == 1) {
            $dias_bps_primera_insersion = 0;
            $dias_bps_segunda_insersion = $can_numero_mes_final - 1;
        }

    // Diferencia de 1 día
    } else if ($cant_dias - $dias_bps == 1) {
        if ($can_numero > 1) {
            $dias_bps_primera_insersion = $can_numero - 1;
            $dias_bps_segunda_insersion = $can_numero_mes_final;

        } else if ($can_numero == 1) {
            $dias_bps_primera_insersion = 0;
            $dias_bps_segunda_insersion = $can_numero_mes_final;
        }
    }
}
```

#### ✔ ¿Qué resuelve este bloque?

- Determina cómo **distribuir correctamente** los días subsidiados por BPS cuando una certificación **cruza de un mes a otro**.  
- Se adapta a los casos donde la diferencia entre días totales y días BPS es de **1, 2 o 3 días**, según normativa.  
- Calcula correctamente **cuántos días cubre BPS en cada período**, evitando inconsistencias en las liquidaciones.  
- Asegura que la parte que la empresa debe cubrir (SEFMU) se calcule sobre la base correcta.

---
✔ Generación de recibos y detalles finales

Este bloque corresponde a la etapa final del proceso, donde el sistema:

Crea el recibo PDF para cada funcionario.

Inserta la información procesada (días cubiertos, días no cubiertos, subsidios, totales).

Genera el detalle final para ser enviado al usuario o archivado dentro del sistema.

📌 Inicialización del PDF y estructura principal
```php
$pdf = new FPDF();
$pdf->AddPage();
$pdf->SetFont('Arial', 'B', 12);

$pdf->Cell(0, 10, "Recibo Subsidio por Enfermedad", 0, 1, 'C');
$pdf->Ln(5);

$pdf->SetFont($default_font, '', 9);
$pdf->Cell($x_ini, $y_text, utf8_decode("Funcionario:"));
      
$x = $x_ini+22;
$pdf->SetXY($x, $y);
$pdf->SetFont('Arial','',10);
$pdf->Cell($x_ini, $y_text,utf8_decode($funcionario_nombre_completo));
$pdf->SetFont($default_font, '', 9);

$x = $x_ini;
$y = $y + 4;
$pdf->SetXY($x, $y);
$pdf->Cell($x_ini, $y_text, utf8_decode("Nº Documento: ".$funcionario->getDocumento()));
```
📌 Periodo certificado y dias computados

```php
$pdf->Cell($x,$y_text,$subs_per['per_desde'].' al '. $subs_per['per_hasta']);
				
$computados = $subs_per['per_dias'] ;
$descontados = $subs_per['per_dias_menos'];

$x = $x+ 70 ;
$pdf->SetXY($x, $y);
$pdf->Cell($x,$y_text,'Computados = '.$computados.' Descontados = '.$descontados);
```
📌Montos finales
```php
$pdf->Cell($x,$y_text,'LIQUIDO A COBRAR');
$x = $x_ini + 184;
$y = $y  ;
$pdf->SetXY($x, $y);
$pdf->Cell($x,$y_text,number_format($liquido_a_cobrar,0,'.', ',').'.00');
```
📌 Salida del archivo
```php
$nombre_pdf = "recibo_" . $cedula . "_" . $periodo . ".pdf";
$pdf->Output('F', "recibos/" . $nombre_pdf);
```
📝 Explicación técnica 

En esta etapa:

Se genera un nuevo PDF empleando FPDF.

Se insertan datos personales del funcionario y del período.

Se arma una tabla clara que muestra:

Días que paga la empresa

Días subsidiados por BPS

Montos correspondientes

Se calcula el total final.

El archivo se exporta a la carpeta configurada en el sistema.

### 📍 Recibo generado para el empleado
![Recibo](Recibo_subsidio.png)



✔  Generación de archivos bancarios (Scotiabank)

Este bloque del sistema se encarga de:

Formatear correctamente los montos (contemplando separadores diferentes según servidor).

Estructurar cada línea con la máscara solicitada por Scotiabank.

Completar con ceros a la izquierda hasta alcanzar el largo requerido por el banco.

Escribir cada línea en el archivo TXT correspondiente.

Preparar la descarga del archivo bancario generado.

📌 Creacion de archivo txt para bancos
```php
foreach ($liquidaciones as $liqui){
    $lineas='';
    
    // Normalizar el valor líquido (punto o coma)
    $liquido = (string)$liqui['liquido'];
    $liquido_dividido = explode(".", $liquido);
    if (!isset($liquido_dividido[1])){
        $liquido_dividido = explode(",", $liquido);
    }
    if (!isset($liquido_dividido[1])){
        $liquido_dividido[1] = '00';
    }
    
    // Unir entero + decimales sin separador
    $liquido = $liquido_dividido[0].$liquido_dividido[1];
    $largo = strlen($liquido);
    $cerosextra = 15 - $largo;
    $cifraconcero = '';
    
    // Completar con ceros hasta 15 caracteres
    for ($i = 0; $i < $cerosextra; $i++) {
        $cifraconcero .= '0';
    }

    // Construcción de la línea según formato 
    $lineas = 
        $cuenta.'           '.'UYJ0307'.$mes.$anio.
        '                     '.$cifraconcero.$liquido.
        '+00'.$liqui['cuenta'];

    fwrite($bank, $lineas."\r\n");
}

fclose($bank);
``` 
📌 Descarga del archivo generado
```php
$filePathscotia = 'TXTLiquidaciones/'.$nombre_archivo;

if(!empty($nombre_archivo) && file_exists($filePaths)){
    header("Cache-Control: public");
    header("Content-Description: File Transfer");
    header("Content-Disposition: attachment; filename=$nombre_archivo");
    header("Content-Type: application/zip");
    header("Content-Transfer-Encoding: binary");

    readfile($filePaths);
}
```

🧾 Explicación técnica del proceso

La generación de archivos bancarios es una etapa clave del sistema, ya que permite automatizar el pago de los subsidios liquidados mediante la creación de archivos TXT con el formato exacto requerido por cada institución bancaria.

En el caso de este banco en particular, el sistema realiza los siguientes pasos:

Normalización del monto líquido
El valor a depositar puede venir con punto o coma como separador decimal, dependiendo del servidor donde corra la aplicación.
El sistema unifica este valor, separa la parte entera y decimal, y luego los concatena sin separador para cumplir con el formato del banco.

Formateo del monto con longitud fija
Los montos deben tener un largo exacto de 15 caracteres, completando con ceros a la izquierda cuando sea necesario.
Este paso es indispensable para que el banco pueda procesar el archivo correctamente.

Construcción de la línea bancaria
Cada línea del archivo TXT incluye:

La cuenta de la institución.

Código fijo requerido por el banco.

Mes y año de liquidación.

Monto formateado a 15 dígitos.

Indicadores bancarios.

Número de cuenta del funcionario.

Escritura del archivo
Cada línea generada se escribe en un archivo .txt que luego es utilizado por la empresa para subir al sistema del banco y ejecutar los depósitos.

Preparación para descarga
Si el archivo fue generado correctamente, el sistema envía los encabezados necesarios para iniciar la descarga automática desde el navegador.

Con este proceso completamente automatizado, la empresa evita errores humanos, acelera el proceso de pago y cumple con los formatos exigidos por las instituciones bancarias.
