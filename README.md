# Sherpa → CRM Automation
**Clipboard-based extraction – Parsing – Field Mapping – Autofill**

Herramienta local para acelerar la campaña de renovación (~4000 pólizas) extrayendo información desde Sherpa (vía RDP) mediante **portapapeles**, convirtiéndola en datos estructurados (**parsing + normalización + validación**) y realizando **autofill** del formulario en **Vtiger** usando automatización de UI (Playwright).  
> Enfoque **no invasivo**: no se instala nada en el remoto. El flujo termina en **human-in-the-loop** (el usuario revisa y guarda manualmente).

---

## Tabla de contenidos
- [Contexto](#contexto)
- [Objetivo](#objetivo)
- [Alcance](#alcance)
- [Fuera de alcance](#fuera-de-alcance)
- [Cómo funciona (overview)](#cómo-funciona-overview)
- [Requisitos](#requisitos)
  - [Requisitos funcionales](#requisitos-funcionales)
  - [Requisitos no funcionales](#requisitos-no-funcionales)
- [Tecnologías](#tecnologías)
- [Instalación](#instalación)
- [Configuración](#configuración)
- [Uso](#uso)
- [Validación y control de calidad](#validación-y-control-de-calidad)
- [Manejo de modales y desplegables](#manejo-de-modales-y-desplegables)
- [Seguridad / PII](#seguridad--pii)
- [Logs y auditoría](#logs-y-auditoría)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Troubleshooting](#troubleshooting)
- [Roadmap](#roadmap)

---

## Contexto
- **Origen:** Sherpa accesible mediante **RDP (Remote Desktop Protocol)**. En RDP la información se presenta como imagen/píxeles (no hay DOM/HTML accesible).
- **Destino:** CRM local **Vtiger**, formulario con secciones (digitador/fecha, póliza, documentos, dirección, titular, cónyuge y dependientes).
- **Restricciones:** no se puede instalar nada en el remoto; el CRM no dispone de importación masiva para todos los campos.

---

## Objetivo
Reducir el tiempo y errores de captura manual durante la renovación, mediante:
- reutilizar la capacidad de **copiar** información del remoto,
- convertir el texto copiado a datos fiables (parsing + normalización + validación),
- autorrellenar Vtiger con automatización controlada,
- mantener revisión humana final (**no auto-guardar**).

---

## Alcance
Incluye:
- Captura por secciones (People / Policy / Address…).
- Parsing tolerante a variaciones mediante **sinónimos** (ej. *Exchange ID* ↔ *MP-ID*).
- Normalización de fechas, importes e identificadores.
- Validación de campos críticos (bloqueo de autofill si faltan o son inválidos).
- Autofill en Vtiger (inputs, selects y modales) **sin tocar pagos/autopay**.
- Soporte para titular + cónyuge + N dependientes (0..N), según el layout del CRM.
- Logs por póliza/ejecución (audit trail).

---

## Fuera de alcance
- No se modifica información de **pagos/autopay** (campos “NO TOCAR”).
- No se realiza **auto-guardado** del formulario (human-in-the-loop siempre).
- OCR no es método principal; solo sería plan B con aprobación.
- Integración por API/importación masiva (no disponible actualmente).

---

## Cómo funciona (overview)
1. El usuario copia (Ctrl+C) una sección de datos desde Sherpa (RDP).
2. Pega el texto en la app local (wizard por pasos) o se lee del clipboard (según configuración).
3. El sistema:
   - **Parsea** el texto y extrae campos usando regex/heurísticas.
   - **Normaliza** moneda/fechas/IDs.
   - **Valida** campos críticos (errores bloquean autofill, warnings requieren revisión).
   - Aplica **field mapping** (sinónimos y equivalencias Sherpa → campo lógico → Vtiger).
4. Se ejecuta **autofill** en Vtiger con Playwright (UI automation).
5. El usuario revisa y pulsa “Guardar” manualmente.

---

## Requisitos

### Requisitos funcionales
- **RF-01** Captura por pasos: permitir pegar secciones (People, Policy, Address opcional).
- **RF-02** Soportar texto con etiquetas y/o formato de tabla (tabs/saltos de línea).
- **RF-03** Diccionario de sinónimos para campos (p.ej. *Exchange ID* = *MP-ID*).
- **RF-04** Modelo interno: titular, cónyuge (si aplica) y dependientes (0..N).
- **RF-05** Normalización: moneda, fechas, IDs y limpieza de texto.
- **RF-06** Validación: reglas duras (bloqueo) y reglas blandas (warning).
- **RF-07** Previsualización: mostrar resumen estructurado antes de autofill.
- **RF-08** Autofill en Vtiger para campos en alcance (inputs/selects/modales).
- **RF-09** Catálogos: mapear valores de Sherpa a opciones de Vtiger (Aplica/No aplica).
- **RF-10** Lista “NO TOCAR”: garantizar que pagos/autopay no se modifiquen.
- **RF-11** Logging: registrar resultado, errores y campos omitidos por póliza.
- **RF-12** Reintento: permitir reejecución sin sobrescribir campos prohibidos y alertar de duplicados.

### Requisitos no funcionales
- **RNF-01 (Seguridad/PII)** Enmascarar/evitar persistir SSN/DOB en logs.
- **RNF-02 (Usabilidad)** Flujo guiado tipo wizard para usuarios no técnicos.
- **RNF-03 (Mantenibilidad)** Configuración externa (JSON) para sinónimos, catálogos y selectores.
- **RNF-04 (Robustez)** Tolerancia a variaciones razonables del texto copiado.
- **RNF-05 (Trazabilidad)** Audit trail por ejecución: timestamp, estado, warnings/errores.

---

## Tecnologías
- **Python 3.10+** (app local)
- **Playwright** (automatización web para Vtiger; no es extensión)
- Regex/validación en Python
- (Opcional) **Tkinter** para interfaz gráfica tipo wizard

> **Playwright** se instala con `pip` y controla el navegador (Edge/Chrome/Chromium) desde Python.

---

## Instalación
> Requisitos previos: Windows, Python instalado y acceso a Vtiger desde navegador.

1) Crear y activar entorno virtual:
```bash
python -m venv .venv
.venv\Scripts\activate

2) Instalar dependencias
pip install -r requirements.txt

3) Instalar navegadores para Playwright:
playwright install

 ****Configuración****

La herramienta usa archivos JSON para evitar hardcode.
Crea/edita estos ficheros:

config/synonyms.json → sinónimos por campo (Exchange ID ↔ MP-ID).
config/catalogs.json → valores controlados para selects/modales.
config/vtiger_selectors.json → selectores (name/id) de campos en Vtiger.
config/do_not_touch.json → campos prohibidos (pagos/autopay).
config/app.json → URL de Vtiger, modo de captura, etc.

Ejemplo:
{
  "mp_id": ["MP-ID", "MP ID", "Exchange ID", "ExchangeID"],
  "premium": ["Premium", "Prima", "Monthly Premium"]
}



