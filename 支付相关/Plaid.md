# Plaid

## 获取access_token

在进入页面时：

1. 调用`/api/info`接口获得：

    ```json
    {
        "item_id": null,
        "access_token": null,
        "products": [
            "assets",
            "auth",
            "identity",
            "investments",
            "liabilities",
            "transactions"
        ]
    }
    ```

    主要是用来得到我们使用的products

2. 调用`/api/create_link_token`接口，获得：

    ```json
    {
        "expiration": "2022-04-14T13:01:32Z",
        "link_token": "link-sandbox-96fa5dce-eac2-4a28-b6b9-1a97aa7589bd",
        "request_id": "ECXOmJY3AXydkBm"
    }
    ```

    主要是用来获取link_token，并在过期时间内使用

3. 通过webview调用`https://cdn.plaid.com/link/v2/stable/link.html?isWebview=true&token="GENERATED_LINK_TOKEN"&receivedRedirectUri=`接口，获得`public_access_token`

    这里需要用户进行操作，在第三方的页面上输入用户名、密码，并进行相关的验证。通过onSuccess或conection回调拿到public_access_token

4. 通过调用`/item/public_token/exchange`，传入`public_access_token`，从而得到：

    ```json
    {
    access_token: 'access-sandbox-022997c5-3b3b-46f8-a5ad-46c577799e33',
    item_id: 'EQRpMEGKr9SJpDexm1LMIRKRGzW7rjCXdMRr3',
    request_id: 'JhQyYPsuEJw5HGa'
    }
    ```

    有了`access_token`和`item_id`，才有了调用其它product对应的产品的能力。这个access_token是可以持久化存储和使用的。

## webhook

Plaid sends POST payloads with raw JSON to your webhook URL from one of the following IP addresses:

- 52.21.26.131
- 52.21.47.157
- 52.41.247.19
- 52.88.82.239

Note that these IP addresses are subject to change.

If there is a non-200 response or **no response within 10 seconds** from the webhook endpoint, Plaid will retry sending the webhook up to **two times** with a few minutes in between each webhook.

可以通过调用`/sandbox/item/fire_webhook`端点来触发啥想环境的webhook。
