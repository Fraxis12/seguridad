<div align="center">

<img src="logo_gbvp.png" alt="Escudo Bomberos France N°3" width="140" />

</div>

---

<h1 align="center">MS-SEGURIDAD-BOMBEROS</h1>

<p align="center"><b>Microservicio de autenticación y control de accesos para la plataforma de gestión documental de Bomberos France N°3.</b></p>

La Compañía trabaja con personal voluntario que rota por sección, y no hay un equipo de TI que esté creando y dando de baja cuentas cada vez que alguien entra o sale. Por eso no se maneja una cuenta por persona, sino una cuenta por sección (5 en total), y el problema real que resuelve este módulo es:

> ¿cómo se controla el acceso y se cambia una contraseña compartida sin depender de un desarrollador cada vez que hace falta?

---

## Cognito

El login no tiene lógica propia: el frontend se conecta directo a Amazon Cognito con Amplify. No construimos nuestro propio manejo de contraseñas ni nada parecido — eso es justo lo que se evita delegándolo a Cognito, en vez de terminar guardando contraseñas o tokens a mano en una base de datos.

User Pool con 5 grupos (uno por rol): `Jefatura`, `Jefe_Administracion`, `Jefe_ServicioGeneral`, `Jefe_Maquinas`, `Jefe_Sanidad`, más un App Client que usa el frontend para el login.

Como son cuentas compartidas y no individuales, no hay endpoint para crear usuarios nuevos: las 5 se crean una sola vez con `scripts/seed_users.py`. Login por username, no por correo:

| Username | Rol |
|---|---|
| `jefatura` | Jefatura |
| `administracion` | Jefe de Administración |
| `serviciogeneral` | Jefe de Servicio General |
| `maquinas` | Jefe de Máquinas |
| `sanidad` | Jefe de Sanidad |

---

## Panel de accesos

Aquí está el riesgo real de tener cuentas compartidas: si la contraseña de una sección se filtra o simplemente se olvida, antes la única salida era pedirle a alguien que entre a la consola de AWS a mano, o terminar pasando la contraseña nueva por WhatsApp sin que quede registro de quién la cambió. Este panel evita eso — le da a Jefatura una forma de resetear la contraseña de cualquier cuenta de sección sin tocar infraestructura ni depender de un desarrollador.

Por el mismo motivo, el acceso está limitado a Jefatura: si cualquier jefe de sección pudiera resetear la contraseña de otra sección (o la de Jefatura), perdería sentido tener roles separados. Por eso Jefatura ve las 4 cuentas de sección pero no puede tocar la suya desde aquí, y nadie más puede entrar a este panel.

Son 2 Lambdas detrás de API Gateway, con un Authorizer de Cognito — sin token válido no entra nadie.

### Endpoints

Base URL (dev):

```
https://3fbn9ktlw8.execute-api.us-east-1.amazonaws.com/dev
```

| Método | Ruta | Quién | Qué hace |
|:---:|---|:---:|---|
| `GET` | `/admin/cuentas` | Jefatura | Lista las 4 cuentas de sección |
| `PATCH` | `/admin/cuentas/{id}/password` | Jefatura | Cambia la contraseña de una cuenta. Body: `{"nueva_password": "..."}` |

Van con header:

```
Authorization: Bearer <IdToken>
```

Si el token no es de Jefatura, devuelve 403.
