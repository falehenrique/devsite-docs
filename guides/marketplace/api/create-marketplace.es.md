# Cómo integrar marketplace via API

> WARNING
>
> Pre-requisitos
>
> * Tener implementado [API](/guides/payments/api/introduction.es.md).

Para comenzar debes:

1. Dar de alta una aplicación de tipo _Marketplace_.
2. Solicitar a tus vendedores que se vinculen.
3. Crear los pagos en nombre de tus vendedores.


## 1. Cómo crear tu aplicación

[Crea tu aplicación](https://applications.mercadopago.com/), marcando la opción **MP Connect / Marketplace mode** y los **scopes** `read`, `write` y `offline_access`.

También debes completar una **Redirect URI** donde serán redireccionados los vendedores para poder ser vinculados correctamente.

Una vez creada, obtendrás el `APP_ID` (identificador de aplicación) necesario para el siguiente paso.

## 2. Vinculación de cuentas

Para operar en Mercado Pago en nombre de tu vendedor, debes primero solicitarle autorización. Para esto, redirige al usuario a la siguiente URL reemplazando en `client_id` el valor de `APP_ID` y la `redirect_uri` que obtuviste en el paso anterior:

`https://auth.mercadopago.com.ar/authorization?client_id=APP_ID&response_type=code&platform_id=mp&redirect_uri=http%3A%2F%2Fwww.URL_de_retorno.com`

Recibirás el código de autorización en la url que especificaste:

`http://www.URL_de_retorno.com?code=AUTHORIZATION_CODE`

Este `AUTHORIZATION_CODE` será utilizado para crear las credenciales, y tiene un tiempo de validez de 10 minutos.

> WARNING
>
> Consejo
>
> Puedes incluir algún parámetro en `redirect_uri` para identificar a qué vendedor corresponde el código de autorización que recibiste, como su _e-mail_, el _ID_ de usuario en tu sistema o cualquier otra referencia útil.


### Crea las credenciales de tus vendedores

Usa el código de autorización, obtenido en el paso anterior, para obtener las credenciales del usuario mediante la API de OAuth y así poder operar en su nombre.  

Request:

```curl
curl -X POST \
     -H 'accept: application/json' \
     -H 'content-type: application/x-www-form-urlencoded' \
     'https://api.mercadopago.com/oauth/token' \
     -d 'client_id=CLIENT_ID' \
     -d 'client_secret=CLIENT_SECRET' \
     -d 'grant_type=authorization_code' \
     -d 'code=AUTHORIZATION_CODE' \
     -d 'redirect_uri=REDIRECT_URI'
```

Los parámetros que debes incluir son:

* `client_id`: El valor de `APP_ID`.
* `client_secret`: Tu `CLIENT_SECRET`.
* `code`: El código de autorización que obtuviste al redirigir al usuario de vuelta a tu sitio.
* `redirect_uri`: Debe ser la misma Redirect URI que configuraste en tu aplicación.


_Response_:

```json
{
    "access_token": "MARKETPLACE_SELLER_TOKEN",
    "public_key": "PUBLIC_KEY",
    "refresh_token": "TG-XXXXXXXXX-XXXXX",
    "live_mode": true,
    "user_id": USER_ID,
    "token_type": "bearer",
    "expires_in": 15552000,
    "scope": "offline_access payments write"
}
```

En la respuesta, además del `access_token` y `public_key` del vendedor que se ha vinculado, obtienes el `refresh_token` que debes utilizar para renovar periódicamente sus credenciales.

> NOTE
>
> Nota
>
> Las credenciales tienen un **tiempo de validez de 6 meses**.


### Renueva las credenciales de tus vendedores

Este proceso debes efectuarlo periódicamente para asegurar tener almacenado en tu sistema credeciales de vendedores que estén vigentes, dado que son válidas por 6 meses.

Sugerimos que si en el flujo de pago obtienes algún error relacionado al _Access Token_ que estás utilizando, refresques automáticamente y reintentes el pago, antes de mostrar un error al comprador.

```curl
curl -X POST \
     -H 'accept: application/json' \
     -H 'content-type: application/x-www-form-urlencoded' \
     'https://api.mercadopago.com/oauth/token' \
     -d 'client_secret= ACCESS_TOKEN' \
     -d 'grant_type=refresh_token' \
     -d 'refresh_token=USER_RT'
```

Respuesta esperada:

```json
{
    "access_token": "MARKETPLACE_SELLER_TOKEN",
    "public_key": "PUBLIC_KEY",
    "refresh_token": "TG-XXXXXXXXX-XXXXX",
    "live_mode": true,
    "user_id": USER_ID,
    "token_type": "bearer",
    "expires_in": 15552000,
    "scope": "offline_access payments write"
}
```

## 3. Integra la API para recibir pagos

Para recibir pagos en nombre de tus vendedores debes integrar la [API](/guides/payments/api/introduction.es.md), utilizando el `access_token` de cada vendedor para tu aplicación.

Si deseas cobrar una comisión por cada cobro que procesa tu aplicación en nombre de tu usuario, sólo debes agregar dicho monto en el parámetro `application_fee` al crear el pago:

[[[
```php
<?php  

  require ('mercadopago.php');
  MercadoPago\SDK::configure(['ACCESS_TOKEN' => 'ENV_ACCESS_TOKEN']);

  $payment = new MercadoPago\Payment();

  $payment->transaction_amount = 100;
  $payment->token = "ff8080814c11e237014c1ff593b57b4d";
  $payment->description = "Title of what you are paying for";
  $payment->installments = 1;
  $payment->payment_method_id = "visa";
  $payment->user_token = "ENV_USER_TOKEN";
  $payment->payer = array(
    "email" => "test_user_19653727@testuser.com"
  );

  $payment->save();

?>
```
```java

import com.mercadopago.*;
MercadoPago.SDK.configure("ENV_ACCESS_TOKEN");

Payment payment = new Payment();

payment.setTransactionAmount(100)
      .setToken('ff8080814c11e237014c1ff593b57b4d')
      .setDescription('Title of what you are paying for')
      .setInstallments(1)
      .setPaymentMethodId("visa")
      .setUserToken("ENV_USER_TOKEN")
      .setPayer(new Payer("test_user_19653727@testuser.com"));

payment.save();

```
```node

var mercadopago = require('mercadopago');
mercadopago.configurations.setAccessToken(config.access_token);

var payment_data = {
  transaction_amount: 100,
  token: 'ff8080814c11e237014c1ff593b57b4d'
  description: 'Title of what you are paying for',
  installments: 1,
  payment_method_id: 'visa',
  user_token: "ENV_USER_TOKEN"
  payer: {
    email: 'test_user_3931694@testuser.com'
  }
};

mercadopago.payment.create(payment_data).then(function (data) {
  // Do Stuff...
}).catch(function (error) {
  // Do Stuff...
});

```
```ruby

require 'mercadopago'
MercadoPago::SDK.configure(ACCESS_TOKEN: ENV_ACCESS_TOKEN)

payment = MercadoPago::Payment.new()
payment.transaction_amount = 100
payment.token = 'ff8080814c11e237014c1ff593b57b4d'
payment.description = 'Title of what you are paying for'
payment.installments = 1
payment.payment_method_id = "visa"
payment.user_token = "ENV_USER_TOKEN"
payment.payer = {
  email: "test_user_19653727@testuser.com"
}

payment.save()

```
]]]


El vendedor va a recibir la diferencia entre el monto total y las comisiones, tanto la de Mercado Pago, como la del _Marketplace_, así como cualquier otro importe que se deba descontar de la venta.

### Notificaciones

Es necesario que envíes tu `notification_url`, donde recibirás aviso de todos los nuevos pagos y actualizaciones de estados que se generen.

Puedes recibir notificaciones cuando tus clientes autoricen o desautoricen tu aplicación, [configurando la URL](https://www.mercadopago.com/mla/account/webhooks) en tu cuenta.

En el artículo de [notificaciones](/guides/notifications/webhooks.es.md) puedes obtener más información.

### Devoluciones y cancelaciones

Las devoluciones y cancelaciones podrán ser realizadas tanto por el _marketplace_ como por el vendedor, vía API o desde la cuenta de Mercado Pago.

En el caso de las cancelaciones, solo podrán ser realizadas  utilizando la API de cancelaciones.

Puedes encontrar más información en el artículo sobre [devoluciones y cancelaciones](/guides/manage-account/refunds-and-cancellations.es.md).
