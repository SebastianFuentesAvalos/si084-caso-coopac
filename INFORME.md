# Informe de auditoría · Core financiero de la COOPAC Santa Rosa

**SI-084 · Auditoría de Sistemas** · Examen práctico de Unidad I

| | |
|---|---|
| **Apellidos y nombres** | Sebastian Nicolas Fuentes Avalos |
| **Código de estudiante** | 2022073902 |
| **URL del repositorio** | https://github.com/SebastianFuentesAvalos/si084-caso-coopac/tree/examen-u1 |
| **Fecha** | 30/09/2026 · Corte de auditoría: 31/12/2025 |

Se evaluó el servidor de base de datos del core financiero contra las siete reglas de la política de seguridad de la COOPAC Santa Rosa. Ninguna de las siete reglas cumple con los criterios revisados. Los resultados se sustentan en las salidas guardadas en `evidencias/` y en la restauración ejecutada en una base independiente llamada `restauracion`.

## 1. Resultados de los procedimientos

| Regla | Resultado, con cifras | ¿Cumple? | Archivo de evidencia |
|---|---|---|---|
| R1 | `sr_bd` publica el puerto 55432 del host hacia el 5432 del contenedor en `0.0.0.0` y `[::]`, todas las interfaces IPv4 e IPv6. No se limita al propio servidor. | No | `evidencias/P1_puertos.txt` |
| R2 | La contraseña del administrador está en texto plano en `docker-compose.yml`, línea 10, y tiene 10 caracteres; el mínimo es 12. | No | `evidencias/P2_credenciales.txt` |
| R3 | La cuenta `app_core` tiene el atributo Superuser, además de `postgres`: 1 cuenta adicional con privilegios prohibidos por R3. | No | `evidencias/P3_roles.txt` |
| R4 | Hay 16 cuentas activas de 10 personas cesadas. El cese más antiguo es 18/12/2015. Además, 22 cuentas activas no tienen documento de responsable; 4 tienen perfil ADMIN. | No | `evidencias/P4_cesados.txt` · `evidencias/P4_genericas.txt` · `evidencias/P4_resumen.txt` |
| R5 | 23 desembolsos superan su umbral y tienen el mismo usuario registrador y aprobador, por S/ 709 370,47, correspondientes a 21 usuarios distintos. | No | `evidencias/P5_segregacion.txt` |
| R6 | `log_connections = off` y `log_statement = none`, configurados en `docker-compose.yml`, línea 11. R6 exige `on` y `mod`. | No | `evidencias/P6_registro.txt` |
| R7 | Último respaldo disponible: 14/11/2025, 47 días antes del corte 31/12/2025. La restauración ejecutada recuperó `empleados` y `usuarios`, pero falta `desembolsos`. El script excluye esa tabla y el registro muestra fallos por falta de espacio desde el 15/11/2025. El paquete no aporta evidencia de pruebas trimestrales anteriores. | No | `evidencias/P7_respaldos.txt` · `evidencias/P7_restauracion.txt` · `evidencias/P7_ejecucion_restauracion.txt` · `evidencias/P7_comparacion_core.txt` · `evidencias/P7_causa.txt` |

La condición SQL aplicada en P5 fue `monto > umbral_aprobacion AND usuario_registra = usuario_aprueba`. La comparación es estrictamente mayor porque R5 se refiere a desembolsos que superan el umbral.

## 2. Hallazgo 1

| Elemento | Contenido |
|---|---|
| Título | Desembolsos por encima del umbral aprobados por su propio registrador |
| Condición | Se identificaron 23 desembolsos por S/ 709 370,47 en los que `monto > umbral_aprobacion` y `usuario_registra = usuario_aprueba`. Intervinieron 21 usuarios distintos. El mayor caso fue D00680, por S/ 161 160,52. Evidencia: `evidencias/P5_segregacion.txt`. |
| Criterio | R5 de `politica/POLITICA-DE-SEGURIDAD.md`: quien registra un desembolso que supera el umbral no puede aprobarlo. Control de referencia: NTP-ISO/IEC 27001:2022, A.5.3 Segregación de funciones. |
| Causa | La definición de `desembolsos` en `bd/01_core.sql` no incluye una restricción que impida la coincidencia de registrador y aprobador cuando se supera el umbral. La carga desde `datos/desembolsos.csv` admite esos registros. El paquete no permite verificar controles adicionales de la aplicación. |
| Efecto | S/ 709 370,47 fueron desembolsados sin la aprobación independiente exigida. Esto expone a la cooperativa a desembolsos no autorizados y a su ocultamiento; el importe representa operaciones expuestas, no una pérdida o fraude demostrado. |
| Recomendación | El Jefe de Sistemas, junto con el responsable de Créditos, debe impedir en la aplicación y en la base la autoaprobación sobre el umbral, tras revisar y tratar los registros existentes. Implementar y probar el control en 15 días calendario. El responsable de Créditos debe revisar los 23 casos en 10 días calendario y documentar su autorización y las acciones correctivas. |

## 3. Hallazgo 2

| Elemento | Contenido |
|---|---|
| Título | Respaldos incompletos y 47 días sin respaldo exitoso al corte |
| Condición | El último archivo disponible es `core_2025-11-14.sql`. Transcurrieron 47 días hasta el 31/12/2025. La restauración en `restauracion` concluyó y recuperó solo `empleados` y `usuarios`; no recuperó `desembolsos`. El core contiene 4 812 desembolsos por S/ 45 124 350,88. Evidencia: `evidencias/P7_respaldos.txt`, `evidencias/P7_ejecucion_restauracion.txt`, `evidencias/P7_restauracion.txt` y `evidencias/P7_comparacion_core.txt`. |
| Criterio | R7 de `politica/POLITICA-DE-SEGURIDAD.md`: respaldo diario completo que incluya desembolsos y prueba de restauración trimestral. Control de referencia: NTP-ISO/IEC 27001:2022, A.8.13 Respaldo de la información. |
| Causa | `respaldos/respaldo.sh` usa `--exclude-table=desembolsos`. Su comentario del 01/11/2025 atribuye al Jefe de Sistemas la decisión de excluir la tabla por su tiempo de respaldo. `respaldos/respaldo.log` registra errores `No space left on device` desde el 15/11/2025 hasta el corte. Evidencia adicional: `evidencias/P7_causa.txt`. |
| Efecto | Ante la pérdida del core, el respaldo probado no permitiría recuperar la tabla que contiene 4 812 desembolsos por S/ 45 124 350,88. Además, las modificaciones de los 47 días posteriores al último respaldo exitoso quedan expuestas a pérdida. Estos valores describen información en riesgo, no pérdidas económicas ya ocurridas. |
| Recomendación | El Jefe de Sistemas debe resolver la capacidad de almacenamiento, retirar la exclusión de desembolsos y generar y verificar un respaldo completo en 48 horas. En 7 días debe establecer alertas por fallos, control de espacio y retención de respaldos. Debe programar pruebas de restauración trimestrales y conservar sus resultados, con una primera prueba completa dentro de esos 7 días. |
