# Informe de auditoría · Core financiero de la COOPAC Santa Rosa

**SI-084 · Auditoría de Sistemas** · Examen práctico de Unidad I  
**Docente:** Dr. Oscar Juan Jiménez Flores  
**Tacna, Perú**

| Dato | Información |
|---|---|
| Apellidos y nombres | Colque Ponce, Sergio |
| Código de estudiante | 2022073503 |
| URL del repositorio | https://github.com/srg-cp/si084-caso-coopac/tree/examen-u1 |
| Fecha del informe | 30/09/2026 |
| Corte de auditoría | 31/12/2025 |

## 1. Resultados de los procedimientos

La evaluación se hizo contra las siete reglas de `politica/POLITICA-DE-SEGURIDAD.md`. Las cifras de cuentas y desembolsos provienen de las consultas al servidor PostgreSQL del paquete. El corte de los respaldos es el 31/12/2025.

| Regla | Resultado, con cifras | ¿Cumple? | Archivo de evidencia |
|---|---|---|---|
| R1 | `sr_bd` publica PostgreSQL en `0.0.0.0:55432`, accesible por todas las interfaces del equipo anfitrión. | No | `evidencias/P1_puertos.txt` |
| R2 | La contraseña del administrador aparece en texto plano en `docker-compose.yml`, línea 10, y tiene 10 caracteres; la política exige 12 o más. | No | `evidencias/P2_credenciales.txt` |
| R3 | `app_core` tiene el atributo `Superuser` además de `postgres`: 1 cuenta adicional con privilegio prohibido. | No | `evidencias/P3_roles.txt` |
| R4 | Hay 16 cuentas activas de 10 personas cesadas; el cese más antiguo fue el 18/12/2015. Otras 22 cuentas activas no tienen documento de una persona responsable; 4 de ellas tienen perfil `ADMIN`. | No | `evidencias/P4_cesados.txt` · `evidencias/P4_genericas.txt` |
| R5 | Con la condición SQL `monto > umbral_aprobacion AND usuario_registra = usuario_aprueba`, aparecen 23 desembolsos por S/ 709 370,47, registrados y aprobados por el mismo usuario; intervienen 21 usuarios distintos. | No | `evidencias/P5_segregacion.txt` |
| R6 | `log_connections` está en `off` y `log_statement` en `none`. Ambos valores se fijan en `docker-compose.yml`, línea 11; no se registran todas las conexiones ni las modificaciones. | No | `evidencias/P6_registro.txt` |
| R7 | El último respaldo disponible es del 14/11/2025: 47 días antes del corte. La restauración en la base `restauracion` recuperó solo `empleados` y `usuarios`; falta `desembolsos`. En el core, esa tabla contiene 4 812 registros por S/ 45 124 350,88. | No | `evidencias/P7_respaldos.txt` · `evidencias/P7_restauracion.txt` |

## 2. Hallazgo 1

| Elemento | Contenido |
|---|---|
| Título | Respaldo desactualizado e incompleto del core financiero |
| Condición | El último archivo de respaldo es `core_2025-11-14.sql`, fechado el 14/11/2025, 47 días antes del corte. La restauración de ese archivo solo creó `empleados` y `usuarios`; omitió `desembolsos`, que en el core tiene 4 812 registros por S/ 45 124 350,88. La tarea terminó con errores por falta de espacio desde el 15/11 hasta el 31/12/2025. Se verificó en `evidencias/P7_respaldos.txt` y `evidencias/P7_restauracion.txt`. |
| Criterio | R7 de la política: respaldo diario completo que incluya `desembolsos` y prueba trimestral de restauración. Control de referencia NTP-ISO/IEC 27001:2022, A.8.13, Respaldo de la información. |
| Causa | `respaldos/respaldo.sh` ejecuta `pg_dump` con `--exclude-table=desembolsos`. Además, `respaldos/respaldo.log` registra `No space left on device` en las ejecuciones posteriores al 14/11/2025. No se encontró un registro de pruebas trimestrales de restauración en el paquete. |
| Efecto | Ante pérdida de la base, el archivo disponible no permitiría recuperar ninguno de los 4 812 desembolsos presentes en el core al corte, por S/ 45 124 350,88. Además, existe una brecha de 47 días sin respaldo nuevo; dentro de ese período se registraron 604 desembolsos por S/ 6 112 291,50. Estas cifras miden información expuesta, no una pérdida confirmada. |
| Recomendación | El Jefe de Sistemas debe liberar o ampliar el espacio de almacenamiento y retirar la exclusión de `desembolsos` de la tarea de respaldo en un plazo de 24 horas. Debe generar un respaldo completo, restaurarlo en una base separada y comparar tablas y cantidades de registros. Después debe supervisar a diario las ejecuciones y documentar una prueba de restauración cada trimestre. |

## 3. Hallazgo 2

| Elemento | Contenido |
|---|---|
| Título | Desembolsos sobre el umbral aprobados por quien los registró |
| Condición | Se identificaron 23 desembolsos de monto superior al umbral de aprobación en los que `usuario_registra = usuario_aprueba`. Suman S/ 709 370,47 y corresponden a 21 usuarios distintos. El mayor es `D00680`, por S/ 161 160,52. La consulta y el detalle figuran en `evidencias/P5_segregacion.txt`. |
| Criterio | R5 de la política: un desembolso que supera el umbral de aprobación no puede aprobarlo quien lo registró. Control de referencia NTP-ISO/IEC 27001:2022, A.5.3, Segregación de funciones. |
| Causa | Los datos de `desembolsos` contienen registros con el mismo usuario en ambas funciones. El esquema de `bd/01_core.sql` define esas columnas sin una restricción que impida esa combinación cuando el monto supera el umbral. |
| Efecto | La misma persona pudo registrar y aprobar 23 operaciones por S/ 709 370,47 sin revisión independiente, lo que eleva el riesgo de errores o desembolsos indebidos. El importe es el monto de operaciones expuestas, no una pérdida confirmada. |
| Recomendación | La Gerencia de Operaciones debe revisar las 23 operaciones y sus documentos de aprobación en un plazo de cinco días hábiles. El Jefe de Sistemas debe impedir en el core que un mismo usuario registre y apruebe desembolsos que superen el umbral, probar el control y dejar evidencia de su puesta en producción dentro de 30 días. |
