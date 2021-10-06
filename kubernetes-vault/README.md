# Выполнено ДЗ № 11

 - [x] Основное ДЗ
 - [ ] Задание со *

## В процессе сделано:
 - Запуще kind кластер из трех worker нод
 - На кластер kind с помощью helm инсталированы consul и vault
```
kubernetes-vault# helm status vault
NAME: vault
LAST DEPLOYED: Mon Oct  4 14:26:50 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
Thank you for installing HashiCorp Vault!

Now that you have deployed Vault, you should look over the docs on using
Vault with Kubernetes available here:

https://www.vaultproject.io/docs/

Your release is named vault. To learn more about the release, try:

  $ helm status vault
  $ helm get manifest vault

```
 - vault инициализирован:

```kubectl exec -it vault-0 -- vault operator init --key-shares=1 --key-threshold=1
Unseal Key 1: GogT3Pa0S+98wzMVF7JL1EkOg8dxy+YSLChhQeOEjyU=

Initial Root Token: s.ZlCBtjJX2f8OQ3z556QEJARc

Vault initialized with 1 key shares and a key threshold of 1. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 1 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated master key. Without at least 1 keys to
reconstruct the master key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum of
existing unseal keys shares. See "vault operator rekey" for more information.
```
 
 - vault распечатан
```kubernetes-vault# kubectl exec -it vault-0 -- vault status
Key             Value
---             -----
Seal Type       shamir
Initialized     true
Sealed          false
Total Shares    1
Threshold       1
Version         1.8.3
Storage Type    consul
Cluster Name    vault-cluster-0f901dc4
Cluster ID      e3428bfe-8812-e816-c2bf-15aaaf60de4e
HA Enabled      true
HA Cluster      https://vault-0.vault-internal:8201
HA Mode         active
Active Since    2021-10-04T12:26:06.933102584Z

kubernetes-vault# kubectl exec -it vault-1 -- vault status
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.8.3
Storage Type           consul
Cluster Name           vault-cluster-0f901dc4
Cluster ID             e3428bfe-8812-e816-c2bf-15aaaf60de4e
HA Enabled             true
HA Cluster             https://vault-0.vault-internal:8201
HA Mode                standby
Active Node Address    http://10.244.3.5:8200

kubernetes-vault# kubectl exec -it vault-2 -- vault status
Key                    Value
---                    -----
Seal Type              shamir
Initialized            true
Sealed                 false
Total Shares           1
Threshold              1
Version                1.8.3
Storage Type           consul
Cluster Name           vault-cluster-0f901dc4
Cluster ID             e3428bfe-8812-e816-c2bf-15aaaf60de4e
HA Enabled             true
HA Cluster             https://vault-0.vault-internal:8201
HA Mode                standby
Active Node Address    http://10.244.3.5:8200
```
 - Логинимся, запрашиваем список авторизаций
```kubectl exec -it vault-0 -- vault login
Token (will be hidden): 
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                s.ZlCBtjJX2f8OQ3z556QEJARc
token_accessor       mIFCokaLEVoxFmknTZlDHedY
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]

kubectl exec -it vault-0 -- vault auth list
Path      Type     Accessor               Description
----      ----     --------               -----------
token/    token    auth_token_f74ae214    token based credentials
```
 - Заведены секреты
```kubectl exec -it vault-0 -- vault read otus/otus-ro/config
Key                 Value
---                 -----
refresh_interval    768h
password            asajkjkahs
username            otus

kubernetes-vault# kubectl exec -it vault-0 -- vault kv get otus/otus-rw/config
====== Data ======
Key         Value
---         -----
password    asajkjkahs
username    otus
``` 
 - Включена авторизация через k8s
```kubectl exec -it vault-0 -- vault auth list
Path           Type          Accessor                    Description
----           ----          --------                    -----------
kubernetes/    kubernetes    auth_kubernetes_acc32670    n/a
token/         token         auth_token_f74ae214         token based credentials
``` 
 - Создан Service Account vault-auth и применили ClusterRoleBinding
 - Записан конфиг для k8s авторизации  
```kubectl exec -it vault-0 -- vault write auth/kubernetes/config token_reviewer_jwt="$SA_JWT_TOKEN" kubernetes_host="$K8S_HOST" kubernetes_ca_cert="$SA_CA_CRT" issuer="https://kubernetes.default.svc.cluster.local"
```
 - Создана политка и роль в vault
 - Проверено как работает авторизация
 - Авторизуемся через vault-agent и получим клиентский токен
 - Через consul-template достанем секрет и положим его в nginx
 - Итог - nginx получил секрет из волта, не зная ничего про волт (index.html)
 - Создан CA на базе vault
 - Прописаны урлы для ca и отозванных сертификатов
 - Создан промежуточный сертификат
 - Прописан промежуточный сертификат в vault
 - Создан и отзван новый сертификаты
```kubectl exec -it vault-0 -- vault write pki_int/issue/example-dot-ru common_name="gitlab.example.ru" ttl="24h"
Key                 Value
---                 -----
ca_chain            [-----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUf7KGqM809jZ/Ln+YY6uplOzQGt4wDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMTEwMDUxNjIyNDVaFw0yNjEw
MDQxNjIzMTVaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAJ8nlAd2nBJo
Mi8B0wqK/cCmtifsVgiJtRLVU4ZqqJ60Uqzw01rJc66E+HZ8ylsN0Hsq8k6236Jp
cAwu/gz6JUAAx7HteegXNJPg7WainSh7zNNmxy6aQv6cS+9fD/zayOrTENdB/2MP
qexm5CHe9Hu2Gs+tD78BxAGZJocnV+JDkG/a/r8EbsS99hdbfzbyg5EMJLsTmeyu
HigmfBqZ+CCB64F3eb5arjsOWMIGdidRtDZeTPt0hYM7Yljngu44Mb1xL+nNJe3x
ctjTgChRn10avvWzDOvdCZDBntjIMstfYpVON8Bn+V/a+rG1RHxCjiTR98TuH3Zz
O7BppKho+UECAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUbWOgZVe91g9XgSrDYf/7dKNfNFUwHwYDVR0jBBgwFoAU
Z71Uo4p065shL/ABMbiiNGOiTHIwNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
Mw3umigPeL5o4eKshiY1DVxYtMWUfoCmeYuahiuzIhj0ciFHDxGK7XPM8HeDpgPW
gPEl9BMd2tbQw/hXb629WZOH4Fmf0gVl3lPoA4ekKR/r5zgd1aYuQ07aaMT8VdXh
IERqB2IEUuMU8YIGzAbZZgjldHkhNn6A6pNdLHhQwFOlY1qHWKEGBy0juYgtaCw/
Lhjw3VSjze6N24Bau0mmvIEDmPWc8KjbFxF8dA64QoR7LN3GgPW4IMOxVdlW2hGj
q8x3utCItzWHTNqMg2yajcb6aGtnt30MTuhf1RUd/1KzSIBr5emahbH99C4HaTOE
6dzAQAEpj2+OoohsnzoCGA==
-----END CERTIFICATE-----]
certificate         -----BEGIN CERTIFICATE-----
MIIDZzCCAk+gAwIBAgIUVIjtO16nsrejIcbBUQbgDXftg3cwDQYJKoZIhvcNAQEL
BQAwLDEqMCgGA1UEAxMhZXhhbXBsZS5ydSBJbnRlcm1lZGlhdGUgQXV0aG9yaXR5
MB4XDTIxMTAwNjEyMTQxOFoXDTIxMTAwNzEyMTQ0OFowHDEaMBgGA1UEAxMRZ2l0
bGFiLmV4YW1wbGUucnUwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDL
J+2GkXOD9LW238WjRoXEeyYtvIYgfDa8mXgE2eRbNdbhdn3DcNkzopZ1Bp6We+eI
rkyqYUgVOh1LlvbYP0mQow3wIV1Zzl+VytVgG87tc+bsIvUK/CJhf9k5hiuMpZ41
+43+2ON2p7b7B+Iy9fXzMfPHgqmtN+YMDLjioy3SC6c8tF0ar16sTalvs9bvyd4D
ZKamef+/+MrFgolK1UH7d2m+pZO4/46AnxrMmi0pI3Vw9iyEm7mxHqn8zvD9WC+w
T6pCky3PO+h/XZgDoz46lZ4vJViZldTC704NGZ9jIwW1rBVl+dJXCf4HR0aRrwvQ
OD1rZHXjaYYe8xYO0HIhAgMBAAGjgZAwgY0wDgYDVR0PAQH/BAQDAgOoMB0GA1Ud
JQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4EFgQULIT5zyp0I4BzpuDD
3q+NH3xna3MwHwYDVR0jBBgwFoAUbWOgZVe91g9XgSrDYf/7dKNfNFUwHAYDVR0R
BBUwE4IRZ2l0bGFiLmV4YW1wbGUucnUwDQYJKoZIhvcNAQELBQADggEBABEYLAux
/7lV/BH2MxlBzVLjZ3ofIjUeSiJVQ8sLyWiw/OChOQok14ZJM2S44ylMcdJLsz5G
Fmu1dg1MFXEMk6U8ByWdTOhq3kJUJe4alcXW43pJJtMua8ASLLEUyL+CQ5PgpLk8
gERdK8RudHM903LHrgkq9QsB4jGlBwyERjgy7w6avoUztbZzOA2eGAYatLq/Tgxs
+HZMlTbtsGDoiSgrBY5s9N9vzvCVHRhrbr+NkZjr1lpD4POXfwAmovQCH6yuvNMF
mgfRwDtTq3CHnaPuAnJ73ynWc5I6TWUjZdNdWAOXZqlNGwLjc6aJjmXenUcj9g5w
a/mzA/yjWUc11Ys=
-----END CERTIFICATE-----
expiration          1633608888
issuing_ca          -----BEGIN CERTIFICATE-----
MIIDnDCCAoSgAwIBAgIUf7KGqM809jZ/Ln+YY6uplOzQGt4wDQYJKoZIhvcNAQEL
BQAwFTETMBEGA1UEAxMKZXhtYXBsZS5ydTAeFw0yMTEwMDUxNjIyNDVaFw0yNjEw
MDQxNjIzMTVaMCwxKjAoBgNVBAMTIWV4YW1wbGUucnUgSW50ZXJtZWRpYXRlIEF1
dGhvcml0eTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAJ8nlAd2nBJo
Mi8B0wqK/cCmtifsVgiJtRLVU4ZqqJ60Uqzw01rJc66E+HZ8ylsN0Hsq8k6236Jp
cAwu/gz6JUAAx7HteegXNJPg7WainSh7zNNmxy6aQv6cS+9fD/zayOrTENdB/2MP
qexm5CHe9Hu2Gs+tD78BxAGZJocnV+JDkG/a/r8EbsS99hdbfzbyg5EMJLsTmeyu
HigmfBqZ+CCB64F3eb5arjsOWMIGdidRtDZeTPt0hYM7Yljngu44Mb1xL+nNJe3x
ctjTgChRn10avvWzDOvdCZDBntjIMstfYpVON8Bn+V/a+rG1RHxCjiTR98TuH3Zz
O7BppKho+UECAwEAAaOBzDCByTAOBgNVHQ8BAf8EBAMCAQYwDwYDVR0TAQH/BAUw
AwEB/zAdBgNVHQ4EFgQUbWOgZVe91g9XgSrDYf/7dKNfNFUwHwYDVR0jBBgwFoAU
Z71Uo4p065shL/ABMbiiNGOiTHIwNwYIKwYBBQUHAQEEKzApMCcGCCsGAQUFBzAC
hhtodHRwOi8vdmF1bHQ6ODIwMC92MS9wa2kvY2EwLQYDVR0fBCYwJDAioCCgHoYc
aHR0cDovL3ZhdWx0OjgyMDAvdjEvcGtpL2NybDANBgkqhkiG9w0BAQsFAAOCAQEA
Mw3umigPeL5o4eKshiY1DVxYtMWUfoCmeYuahiuzIhj0ciFHDxGK7XPM8HeDpgPW
gPEl9BMd2tbQw/hXb629WZOH4Fmf0gVl3lPoA4ekKR/r5zgd1aYuQ07aaMT8VdXh
IERqB2IEUuMU8YIGzAbZZgjldHkhNn6A6pNdLHhQwFOlY1qHWKEGBy0juYgtaCw/
Lhjw3VSjze6N24Bau0mmvIEDmPWc8KjbFxF8dA64QoR7LN3GgPW4IMOxVdlW2hGj
q8x3utCItzWHTNqMg2yajcb6aGtnt30MTuhf1RUd/1KzSIBr5emahbH99C4HaTOE
6dzAQAEpj2+OoohsnzoCGA==
-----END CERTIFICATE-----
private_key         -----BEGIN RSA PRIVATE KEY-----
MIIEowIBAAKCAQEAyyfthpFzg/S1tt/Fo0aFxHsmLbyGIHw2vJl4BNnkWzXW4XZ9
w3DZM6KWdQaelnvniK5MqmFIFTodS5b22D9JkKMN8CFdWc5flcrVYBvO7XPm7CL1
CvwiYX/ZOYYrjKWeNfuN/tjjdqe2+wfiMvX18zHzx4KprTfmDAy44qMt0gunPLRd
Gq9erE2pb7PW78neA2Smpnn/v/jKxYKJStVB+3dpvqWTuP+OgJ8azJotKSN1cPYs
hJu5sR6p/M7w/VgvsE+qQpMtzzvof12YA6M+OpWeLyVYmZXUwu9ODRmfYyMFtawV
ZfnSVwn+B0dGka8L0Dg9a2R142mGHvMWDtByIQIDAQABAoIBAHsRXAWaFUVJt87p
rhtj/GLzS0SHoSUKukn0Gk2uBXTvn5WDp1n/AcUS6FxKP0XgF3moRQ8t0XHye46w
DCch55mz/RyLybY+m47tiecn0WntPWWtI46dAOLZhSkgyz7vkXEYS4OntdvKa8GU
nAXNFEpX51rkH4+sfjKsfk/lFDvZ+A5CtWSgWu6/Fb6a8qDj6HCwYqSBvrfgWcdI
IIKl6ZVeMWaVxL0nR2DQT0+OrHR+hDeUtU4V09/H2qf6hC1Y129PKCRR1QNWxGPS
3jO3mZzOHQmCyyWRkn3Jbpljwx6oeL3HXg6pYV3K/ndosISh1iDbsc56n5Z+G+hI
AnPkQJUCgYEA+FIyPq9isK0unt/B4MrubelwyvxNDrQdVp6ztrXA3/VLNinlElLU
L2pj14ZQGAyhjXpWRaE6xgw0rzcFQ+j1d87AlTw5WdWENzDU9F71iuP/J9ITfRkB
EkG7VzBF976i1sTsUM/hMQ8yWrf4uKmkIw8ax0Z3bbT8OjSTBRD2ym8CgYEA0XAv
+9+8fospxJT7sl/a0RW8kAcukKk7SFuX7i6dcBrspkbRZCaLf24v3I/h/tuHdBNg
q+1cE8CXZq8vjNo3XlIxOB6Jy0aWhLNlE8YRjiR0arh0afdqsqIkEScvYjfyqYGW
fnmPYuZN9fKEKJ9ZikOvksyGVmWz7xqj/KriFG8CgYBmtYDIwrw8PXVyCzTi6KzT
02Fu5ApvUXptEHle0jBzsb6pKYzxFkdjUUr4ozpPqDHOFdLHPBfWQMgtzMElxJ57
Lo4ja+SAzrrAJTd/2CMRjppD+zVKYeQ6i+uT9YiLH1O1J4BjMIiBRTrboQqEPs6A
HchCslfFjb1hycshplGdiwKBgBNjcl272aRRV72GGULrEsO2Ym1m7M2hjQZmzErV
b+e35l6CQdImq1VRqwadH0vLoN+DB7kC0TpW4u0znJBKh0OpEjtiwFjcIQUJ4nqR
JIDnKQvUJZrFt8/vqK0Z1o4eJc3BXGA6+qYqMd9p4wgrsEtXdsJ9QpZu9dhVvAag
/yrrAoGBAN6T0hp/Zb2jeil33TvcdvFUOTfF8VB7dgJ2fpOqHZzYecnsvRbAwY0P
KX01EEXcGepLHdJYNUNYAmcX9l9b8hpz8yyCKzf+uBF7ipL9TV0+sF7uVZH53ia6
KHuq8TDsCkHax+5lGkQOyQVEkqa7b8Glm/aCcSDKo/x7WleA0hjK
-----END RSA PRIVATE KEY-----
private_key_type    rsa
serial_number       54:88:ed:3b:5e:a7:b2:b7:a3:21:c6:c1:51:06:e0:0d:77:ed:83:77
```

## Ответы на вопросы:
 - Выражение sed 's/\x1b\[[0-9;]*m//g' - удаляет управляющие escape-символы, делающие гиперссылки кликабельными в ssh консоли.
 - Для otus-rw/conﬁg1 явно policy не заданы в отличии от otus-rw/conﬁg. При этом pod vault-0 за'login'ился под root token'ом, имеющим максимальные привилегии. После добавления capabilities - "update", запись в otus-rw/conﬁg становится возможной.

## Как запустить проект:
 - kind create cluster --config kind-config.yaml
 - Следовать инструкциям ДЗ

## PR checklist:
 - [x] Выставлен label с темой домашнего задания
