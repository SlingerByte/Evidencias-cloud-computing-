# Automatización de pedidos con AWS

## Sobre el proyecto

Este proyecto consiste en un sistema que recibe pedidos por medio de una API y los procesa automáticamente utilizando diferentes servicios de AWS.

Cuando llega un pedido, se genera un identificador único y se envía un evento a EventBridge. A partir de ahí, Step Functions se encarga de ejecutar los pasos necesarios: validar la información, revisar el inventario, simular el pago y, si todo sale bien, confirmar el pedido y generar un registro de auditoría.

El proyecto se desarrolló como ejercicio práctico para trabajar con servicios serverless de AWS y entender cómo se pueden conectar para automatizar un proceso.

## Arquitectura

Para construir el sistema se utilizaron los siguientes servicios:

- **Amazon API Gateway:** recibe los pedidos mediante una solicitud HTTP.
- **AWS Lambda:** ejecuta las funciones encargadas de cada parte del proceso.
- **Amazon EventBridge:** recibe el evento generado cuando llega un pedido y activa el flujo de procesamiento.
- **AWS Step Functions:** controla el orden en que se ejecutan las funciones y qué sucede cuando alguna condición no se cumple.
- **AWS IAM:** permite configurar los permisos necesarios para que los servicios puedan comunicarse.
- **Amazon CloudWatch:** permite consultar los registros de ejecución de las funciones Lambda.

El diagrama completo de la arquitectura, con los servicios, sus conexiones y las diferentes rutas del proceso, se encuentra en el documento **Informe Técnico – Automatización de Procesamiento de Pedidos con AWS Serverless**.

## ¿Cómo funciona?

1. El cliente envía un pedido mediante una solicitud `POST` a API Gateway.
2. API Gateway envía la información a la función `receive-order-emilson`.
3. Esta función genera un identificador único para el pedido y agrega la fecha y hora de recepción.
4. El pedido se publica en EventBridge como un evento con el origen `demo.pedidos` y el tipo `NuevoPedido`.
5. EventBridge identifica el evento mediante la regla `order-events-rule-emilson` y activa la máquina de estados de Step Functions.
6. Step Functions ejecuta las funciones Lambda encargadas de validar el pedido, revisar el inventario y simular el pago.
7. Si todas las condiciones se cumplen, espera cinco segundos, confirma el pedido y ejecuta la función de auditoría.

La respuesta de la API se devuelve después de publicar el evento en EventBridge. Esto significa que el cliente no tiene que esperar a que termine todo el procesamiento para recibir una confirmación.

## Funciones Lambda

Se crearon seis funciones Lambda en Python 3.11:

| Función | ¿Qué hace? |
|---|---|
| `receive-order-emilson` | Recibe el pedido, genera su identificador y lo envía a EventBridge. |
| `validate-order-emilson` | Comprueba que el cliente, el producto, la cantidad y el total tengan valores válidos. |
| `check-inventory-emilson` | Revisa si la cantidad solicitada está dentro del límite disponible. |
| `process-payment-emilson` | Simula la aprobación o el rechazo del pago. |
| `confirm-order-emilson` | Cambia el estado del pedido a `CONFIRMED`. |
| `register-audit-emilson` | Genera un resumen con la información y el resultado del pedido. |

## Reglas utilizadas

Para las pruebas se definieron algunas condiciones sencillas:

| Validación | Regla |
|---|---|
| Cliente y producto | Deben ser textos y no pueden estar vacíos. |
| Cantidad | Debe ser un número mayor que cero. |
| Total | Debe ser un número mayor que cero. |
| Inventario | Se considera disponible cuando la cantidad es igual o inferior a 5. |
| Pago | Se aprueba cuando el total es igual o inferior a 1000. |

Los límites de inventario y pago se utilizan únicamente para simular las decisiones del sistema. No hay conexión con un inventario real ni con una pasarela de pagos.

### Posibles resultados

Dependiendo de la información recibida, el flujo puede terminar en alguno de estos estados:

- `SuccessState`: el pedido pasó todas las validaciones y fue confirmado.
- `InvalidOrderFail`: la información del pedido no cumple las condiciones.
- `OutOfStockFail`: la cantidad solicitada supera el límite de inventario.
- `PaymentRejectedFail`: el total del pedido supera el límite establecido para la simulación del pago.

La función de auditoría solo se ejecuta cuando el pedido llega a la etapa de confirmación. Los pedidos rechazados no pasan por esa función.

## Recursos creados en AWS

Los recursos se desplegaron en la región **US East (Ohio) — `us-east-2`**.

| Servicio | Recurso |
|---|---|
| API Gateway | `orders-api-emilson` |
| Ruta HTTP | `POST /pedido` |
| Lambda de recepción | `receive-order-emilson` |
| Regla de EventBridge | `order-events-rule-emilson` |
| Máquina de estados | `order-workflow-emilson` |
| Runtime | Python 3.11 |

La máquina de estados utiliza el tipo Standard de Step Functions.

## Pruebas realizadas

Para comprobar el funcionamiento se enviaron pedidos desde PowerShell y se revisaron las ejecuciones en Step Functions.

### 1. Pedido aprobado

Información enviada:

```json
{
  "cliente": "Johan",
  "producto": "Laptop",
  "cantidad": 1,
  "total": 900
}
```

**Resultado:** el pedido pasó las validaciones, el inventario estaba disponible, el pago fue aprobado y el pedido terminó confirmado. La auditoría también se ejecutó correctamente.

Estado final: `SuccessState`.

### 2. Inventario insuficiente

Información enviada:

```json
{
  "cliente": "Johan",
  "producto": "Laptop",
  "cantidad": 6,
  "total": 900
}
```

**Resultado:** el pedido pasó la validación inicial, pero la cantidad solicitada superó el límite de inventario.

Estado final: `OutOfStockFail`.

### 3. Pago rechazado

Información enviada:

```json
{
  "cliente": "Johan",
  "producto": "Laptop",
  "cantidad": 1,
  "total": 1200
}
```

**Resultado:** el pedido pasó la validación y la revisión de inventario, pero el pago fue rechazado por superar el límite establecido.

Estado final: `PaymentRejectedFail`.

Las tres pruebas se ejecutaron correctamente y sus resultados se comprobaron en el historial de Step Functions.

## Cómo probar la API

La ruta utilizada para recibir los pedidos es:

`POST https://<api-id>.execute-api.us-east-2.amazonaws.com/pedido`

Para enviar un pedido desde PowerShell:

```powershell
$body = @{
    cliente = "Johan"
    producto = "Laptop"
    cantidad = 1
    total = 900
} | ConvertTo-Json -Compress

Invoke-RestMethod `
    -Uri "https://<api-id>.execute-api.us-east-2.amazonaws.com/pedido" `
    -Method POST `
    -ContentType "application/json" `
    -Body $body
```

Reemplaza `<api-id>` por el identificador de tu API Gateway.

Si el pedido se recibe y se publica correctamente, la API devuelve un mensaje de confirmación junto con el `orderId` generado. Para conocer el resultado del procesamiento hay que consultar la ejecución correspondiente en Step Functions.

## Limitaciones y posibles mejoras

Esta versión está pensada como una demostración del funcionamiento de los servicios de AWS. Hay algunas cosas que podrían mejorarse en una versión futura:

- Guardar los pedidos en una base de datos para consultar su información posteriormente.
- Consultar un inventario real en lugar de utilizar un límite fijo.
- Integrar una pasarela de pagos.
- Guardar los registros de auditoría para conservar un historial de los pedidos.
- Agregar autenticación a la API para controlar quién puede enviar solicitudes.
- Enviar notificaciones cuando un pedido sea confirmado o rechazado.

Actualmente, la API no tiene autenticación y la auditoría genera un resumen, pero no lo almacena de forma permanente.

## Limpieza de recursos

Al terminar el proyecto y guardar las evidencias, se deben eliminar los recursos creados en AWS para evitar costos innecesarios.

Entre ellos:

- API Gateway.
- Regla y destino de EventBridge.
- Máquina de estados de Step Functions.
- Funciones Lambda.
- Roles y políticas IAM creados para el proyecto.
- Grupos de registros de CloudWatch relacionados.

## Conclusión

Con este proyecto pude poner en práctica la conexión entre varios servicios de AWS para automatizar el procesamiento de pedidos.

El sistema recibe una solicitud, genera un evento y ejecuta un flujo que permite validar la información, revisar el inventario, simular un pago y confirmar el pedido cuando se cumplen las condiciones.

También permitió trabajar con funciones Lambda, reglas de EventBridge y máquinas de estados de Step Functions en un mismo ejercicio.