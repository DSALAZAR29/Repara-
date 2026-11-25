<div align="center">

# REPARA+

### Sistema BPM de gestión de solicitudes y seguimiento de servicio técnico a domicilio

**Tecno Soluciones S.A.S.** (empresa simulada · Manizales) · Proyecto integrador de **Desarrollo de Sistemas Empresariales**

Universidad de Caldas · Ingeniería Informática · Docente: Diego Franco · **Equipo XX** · Noviembre de 2026

</div>

---

## Tabla de contenido
1. [Resumen ejecutivo](#1-resumen-ejecutivo)
2. [Problema y objetivos](#2-problema-y-objetivos)
3. [Proceso As-Is y To-Be](#3-proceso-as-is-y-to-be)
4. [Requerimientos](#4-requerimientos)
5. [Arquitectura de software](#5-arquitectura-de-software)
6. [Modelo de datos](#6-modelo-de-datos)
7. [Roles, seguridad y trazabilidad](#7-roles-seguridad-y-trazabilidad)
8. [Plan de trabajo y próximos pasos](#8-plan-de-trabajo-y-próximos-pasos)
9. [Estructura del repositorio](#9-estructura-del-repositorio)
10. [Cumplimiento de los entregables](#10-cumplimiento-de-los-entregables)
11. [Supuestos y limitaciones](#11-supuestos-y-limitaciones)
12. [Equipo](#12-equipo)

---

## 1. Resumen ejecutivo

Tecno Soluciones S.A.S. —unas 20 personas y cerca de 600 servicios al mes— registra, asigna y cierra sus órdenes de servicio técnico en **Excel, WhatsApp y registros manuales**, sin una fuente única de verdad. **REPARA+** centraliza el flujo completo: solicitud con ticket único, asignación sugerida por zona y agenda, atención en campo desde una app instalable que funciona **sin conexión**, cierre con firma digital del cliente, bandeja de facturación y trazabilidad de cada cambio.

> **Estado del repositorio:** contiene el análisis, el diseño y la documentación del proyecto (As-Is, To-Be, requerimientos, arquitectura, modelo de datos y sustentación). La construcción del sistema está planificada por fases (ver [§8](#8-plan-de-trabajo-y-próximos-pasos)).

| Indicador | As-Is | To-Be / meta |
|---|---|---|
| Ciclo de la solicitud a la orden lista para facturar | 24,5 h | ≤ 3,5 h (−85,7 %) |
| Pasos del proceso | 10 | 6 |
| Fuentes de datos | 3 (Excel, WhatsApp, papel) | 1 (base de datos única) |
| Asignación de técnico | 2–6 h | < 10 min |
| Errores de transcripción | ≈ 12 por semana | 0 |
| Órdenes cerradas en sitio | — | > 95 % |

*Las cifras provienen de una empresa simulada y se presentan como metas de línea base, por validar con un piloto.*

## 2. Problema y objetivos

**Causa raíz:** no existe un flujo centralizado del registro al cierre. Los duplicados, los cruces de agenda y el cierre en lote (con facturación al día siguiente) son síntomas de ese único problema.

Objetivos medibles (SMART) de la línea base:
1. Reducir la asignación de técnico de 2–6 h a menos de 10 minutos.
2. Cerrar más del 95 % de las órdenes desde el móvil, en el domicilio.
3. Llegar a cero errores de transcripción.
4. Reducir el ciclo hasta la factura de 24,5 h a un máximo de 3,5 h.
5. Aprobar los 8 requisitos obligatorios (MUST) con su criterio de aceptación en la semana 5.

**Fuera del alcance:** facturación electrónica ante la DIAN, inventario de repuestos y pagos en línea.

## 3. Proceso As-Is y To-Be

| Proceso actual (As-Is) | Proceso propuesto (To-Be) |
|---|---|
| ![As-Is](anexos/diagramas/AsIs.png) | ![To-Be](anexos/diagramas/ToBe.png) |

Principio de diseño del To-Be: **sin ticket no hay orden**. El dato nace correcto una sola vez y fluye hasta la facturación.

## 4. Requerimientos

- **15 requisitos funcionales** (RF01–RF15) y **7 no funcionales** (RNF01–RNF07), con criterios de aceptación y priorización MoSCoW: **8 MUST, 11 SHOULD y 3 COULD**.
- **8 historias de usuario** (HU01–HU08) y **6 casos de uso más autenticación** (CU-00 a CU-06), con trazabilidad desde el hallazgo hasta el caso de uso.
- Actores: cliente, asesor de atención, coordinador de despacho, técnico de campo, auxiliar de facturación y gerente.

![Diagrama de casos de uso](anexos/diagramas/Casos_de_uso.png)

## 5. Arquitectura de software

**Monolito modular** con seis módulos (identidad y acceso, solicitudes y órdenes, despacho, campo y sincronización, facturación y reportes, notificaciones), proporcional a la escala de la empresa y más económico de construir y mantener que una plataforma comercial de unos USD 45.000.

| Capa | Tecnología |
|---|---|
| Web de oficina | React + Vite (TypeScript); incluye la consulta pública del cliente |
| App del técnico | PWA instalable offline-first (Service Worker, IndexedDB con Dexie, cola local con UUID) |
| Backend | Node.js + NestJS, API REST, JWT y control de acceso por rol (RBAC) |
| Base de datos | PostgreSQL 16 (Prisma); cola de tareas con pg-boss, sin Redis ni broker adicional |
| Reglas compartidas | Paquete TypeScript (esquemas Zod y máquina de estados) en un monorepo pnpm |
| Despliegue | Contenedores Docker; proxy inverso Caddy con HTTPS automático |
| Mensajería | Web Push, correo y SMS tras adaptadores intercambiables; WhatsApp Business en fase posterior |

Vistas C4 (contexto, contenedores y componentes):

![C4 nivel 1 — Contexto](anexos/diagramas/c1.png)
![C4 nivel 2 — Contenedores](anexos/diagramas/c2.png)
![C4 nivel 3 — Componentes del backend](anexos/diagramas/c3.png)

## 6. Modelo de datos

**16 entidades** alrededor de la orden de servicio. La base de datos hace cumplir las reglas críticas:
- **Consecutivo único:** el número de orden sale de una secuencia anual.
- **Sin cruces de agenda:** una restricción de exclusión sobre la franja horaria por técnico.
- **Trazabilidad:** cada cambio de estado escribe su historial en la misma transacción.
- **Idempotencia:** cada comando enviado desde el celular lleva un UUID para no procesarse dos veces.

![Modelo entidad–relación](anexos/diagramas/er.png)

## 7. Roles, seguridad y trazabilidad

- **Autenticación:** contraseñas con hash argon2, JWT de corta duración con token de renovación, bloqueo tras 5 intentos fallidos.
- **Autorización:** guarda de rol en cada endpoint; el técnico solo ve sus órdenes.
- **Transporte:** HTTPS en todo el tráfico.
- **Consulta pública:** número de orden y teléfono, datos mínimos y límite de intentos.
- **Trazabilidad:** historial de estados (quién, cuándo, de qué estado a cuál y por qué), reasignaciones con motivo y accesos denegados registrados.
- **Disponibilidad:** objetivo ≥ 99 % en el horario de operación y copias de seguridad diarias.

La matriz completa de roles y permisos está en el capítulo 5.3 del PDF consolidado.

## 8. Plan de trabajo y próximos pasos

| Fase | Semanas | Contenido |
|---|---|---|
| 1. Núcleo de registro y despacho | 1–3 | Registro validado, asignación sin cruces, roles y auditoría |
| 2. Operación en campo | 4–5 | App del técnico, cierre en sitio y sincronización |
| 3. Complementos y piloto simulado | 6–7 | Consulta del cliente, bandeja de facturación y medición de indicadores |
| 4. Cierre | 8 | Versión congelada, manual de usuario y sustentación |

**Acciones previas:** verificar los celulares Android de los técnicos, definir «técnico disponible» y confirmar las zonas de cobertura.
**Backlog posterior:** DIAN, inventario de repuestos, pagos en línea y WhatsApp Business.

## 9. Estructura del repositorio

```text
.
├── README.md
├── Proyecto_EquipoXX_DesarrolloSistemasEmpresariales.pdf   # documento consolidado
└── anexos/
    ├── talleres_originales/   # Talleres 1, 3, 4, 5, 6 y 7 (.docx)
    ├── arquitectura/          # Justificación de la arquitectura y complemento (datos, seguridad, C4)
    ├── diagramas/             # As-Is, To-Be, casos de uso, ER y C4 (PNG y fuentes .dot)
    └── sustentacion/          # Presentación ejecutiva y guion de la sustentación
```

Los diagramas del modelo de datos y de la arquitectura tienen su fuente en Graphviz (`.dot`) y se regeneran con `dot -Tpng archivo.dot -o archivo.png`.

## 10. Cumplimiento de los entregables

| Entregable exigido | Dónde está |
|---|---|
| Portada | PDF consolidado, p. 1 |
| Análisis As-Is | PDF, capítulo 2 · `anexos/talleres_originales/` (Taller 3) |
| Diseño To-Be | PDF, capítulo 3 · Taller 4 |
| Requerimientos (RF, RNF, casos de uso e historias) | PDF, capítulo 4 · Talleres 5 y 6 |
| Arquitectura: justificación, modelo de datos, roles, seguridad y trazabilidad | PDF, capítulo 5 · `anexos/arquitectura/` |
| Conclusiones y próximos pasos | PDF, capítulo 6 |
| Sustentación (12–15 min + 5 de preguntas) | `anexos/sustentacion/` |

## 11. Supuestos y limitaciones

- Las cifras de impacto provienen de una empresa simulada; deben validarse con un piloto.
- En la primera versión los avisos al cliente salen por correo y SMS; WhatsApp Business queda para una fase posterior (el RF05 del Taller 5 lo contempla como canal).
- La matriz de roles, la administración de usuarios y la geocodificación de direcciones son propuestas por validar con la empresa.
- La disponibilidad de celulares Android 10 o superior y la definición de las zonas de cobertura siguen por confirmar.
- Los talleres originales se conservan tal como fueron entregados, con su redacción original.

## 12. Equipo

| Integrante | Código | Parte en la sustentación |
|---|---|---|
| Jhon Alejandro Franco Gómez | 13167 | Problema, contexto y objetivos |
| Sebastian Gallego Loaiza | 566 | Procesos As-Is y To-Be |
| Juan David Grisalez Ortega | 47568 | Requisitos y arquitectura C4 |
| Daniel Salazar Castañeda | 41140 | Modelo de datos, integración, valor y cierre |

---

<div align="center">
<sub>Proyecto académico · Desarrollo de Sistemas Empresariales · Universidad de Caldas · Uso con fines educativos</sub>
</div>
