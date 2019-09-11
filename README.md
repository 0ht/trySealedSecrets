# SealedSecretsの試行
Secretを、SealedSecretに暗号化。SealedSecretは、ターゲットのクラスター内で動作するControllerのみが複合化可能。

## 概要
Sealed Secretsは、2つのパーツで構成される。
- クラスター側のController / Operator
- クライアント側のユーティリティ / kubeseal

kubesealは、controllerのみが複合可能な非対称暗号方式でsecretwを暗号化する。
これらの暗号化されたsecretは、`SealedSecret`リソースにエンコードされる。

以下のような感じ。
```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mysecret
  namespace: mynamespace
spec:
  encryptedData:
    foo: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq.....
```

これが、復号化されると、以下の様なSecretと等価になる。
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
  namespace: mynamespace
data:
  foo: bar  # <- base64 encoded "bar"
```

### secretのテンプレートとしての利用


### 公開鍵/証明書
key certificate (public key portion) は、secretをsealする際に使用され、kubesealが実行される箇所で参照できている必要がある。証明書自体は秘匿すべき情報では無いので、正しいものが使われていることを確認できればよい。

kubesealは、実行時にcontrollerから証明書を取得する。（よって、KubernetesAPIかサーバーにセキュアにアクセスできる必要がある）

## 導入
[こちら](https://github.com/bitnami-labs/sealed-secrets/releases)を参照。

### クライアント側
Linux:

```bash
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.8.3/kubeseal-linux-amd64 -O kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```
Mac:
```
brew install kubeseal
```

### クラスター側
kube-systemネームスペースにcontrollerとしてSealedSecretを導入。

以下を実行
```
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.8.3/controller.yaml
```

```
(⎈ |tomcluster:default)eb82649:02_Maven eb82649@jp.ibm.com$ kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.8.3/controller.yaml
service/sealed-secrets-controller created
role.rbac.authorization.k8s.io/sealed-secrets-service-proxier created
rolebinding.rbac.authorization.k8s.io/sealed-secrets-controller created
role.rbac.authorization.k8s.io/sealed-secrets-key-admin created
serviceaccount/sealed-secrets-controller created
customresourcedefinition.apiextensions.k8s.io/sealedsecrets.bitnami.com created
clusterrolebinding.rbac.authorization.k8s.io/sealed-secrets-controller created
clusterrole.rbac.authorization.k8s.io/secrets-unsealer created
deployment.apps/sealed-secrets-controller created
rolebinding.rbac.authorization.k8s.io/sealed-secrets-service-proxier created
```

## 利用方法

```
# json/yaml-encoded secretを作成する
# (--dry-runでの実行。これは単なるlocalファイルであることに留意))
$ kubectl create secret generic mysecret --dry-run --from-literal=foo=bar -o json >mysecret.json

# ここが重要なポイント
# 以下のコマンドで、通常のsecretファイルを暗号化してsealed secretを作成する
$ kubeseal <mysecret.json>mysealedsecret.json

# mysealedsecret.jsonは暗号化され、githubにuploadしてもOK。
# そして、sealed secretをクラスターに適用
$ kubectl create -f mysealedsecret.json

# Profit!
$ kubectl get secret mysecret
```

実行結果
```
#実際にsecretを生成
(⎈ |tomcluster:default)eb82649:02_Maven eb82649@jp.ibm.com$ kubectl create secret generic mysecret --dry-run --from-literal=foo=bar -o json >mysecret.json

# 内容は以下
(⎈ |tomcluster:default)eb82649:02_Maven eb82649@jp.ibm.com$ cat mysecret.json
{
    "kind": "Secret",
    "apiVersion": "v1",
    "metadata": {
        "name": "mysecret",
        "creationTimestamp": null
    },
    "data": {
        "foo": "YmFy"
    }
}

# このsecretを元に、sealed secretを生成する。
(⎈ |tomcluster:default)eb82649:02_Maven eb82649@jp.ibm.com$ kubeseal <mysecret.json >mysealedsecret.json

# 中身は以下の通り。暗号化されてる。
(⎈ |tomcluster:default)eb82649:02_Maven eb82649@jp.ibm.com$ cat mysealedsecret.json 
{
  "kind": "SealedSecret",
  "apiVersion": "bitnami.com/v1alpha1",
  "metadata": {
    "name": "mysecret",
    "namespace": "default",
    "creationTimestamp": null
  },
  "spec": {
    "template": {
      "metadata": {
        "name": "mysecret",
        "namespace": "default",
        "creationTimestamp": null
      }
    },
    "encryptedData": {
      "foo": "AgAjcK7F7+kkGfyGqYKr1zVpF645TFKPP5gCjtVaryAC0+sRqKnFvbNAeg3JJzUCggoxBj54nUhQ+4s9FWe20y8SHZHScBeogZqWx583j8B89vgabyz6zBkFFXyE65sB3u0TCxcZIYHejhSXL6TQP9e4J+wRadJNsn7xUd2mDGB3l92C3fAVI3T2AnuTTnEMhxluJKrhoCnpBb2EOtaI7Y8ustZOaO6i3mzVwC1bQKC25yRbzNmPht/xB77sIp4XKWtPd4ED22VhTQ3JTXYL7GSMHXZppJqIRXZHwUv5+gmaM5BUkWQ6CRzb7RzoFMyzRdIHzp/IbETlJLTasKbtHsehDjyS8Rd+euCt7Rkwj4O/J8WwvPPNAjLTFglVVuKJHsssKB7JxMDvVzXDShxqlysLmv4tWTpxnD45ZISbmJ57x5kwC+CR/KQwvhrridvMYEtUsHV8+gjwd7/JY8BAhL2zpjXagw5GtqOhKj7Sqd1WTyeW196fZqh+B3cEzzSM0RlTSKJ7p56QOedCamJ82VaQiZ2FUdm7eg8ZXCR4yw9C6POmiwoNNMSe/ECV95bYa3BKh+ZgRiuLp63QrCbjlcm8crqlGZ071FZyydAFXNDaKFkcoV8HzwKeA2iYS5muAQSiFiTg0SYgwkpZm//HHsYED9ck0pZPP0A4BRbS+7vBAtUFqsk1+HpnrJNV2PI06h93N4w="
    }
  },
  "status": {
    
  }
}

# クラスターに適用する
(⎈ |tomcluster:default)eb82649:02_Maven eb82649@jp.ibm.com$ kubectl apply -f mysealedsecret.json 
sealedsecret.bitnami.com/mysecret created

# 適用後は通常のsecretと同じ様に扱われる。
(⎈ |tomcluster:default)eb82649:02_Maven eb82649@jp.ibm.com$ kubectl get secret mysecret -o yaml
apiVersion: v1
data:
  foo: YmFy
kind: Secret
metadata:
  creationTimestamp: "2019-09-11T08:41:36Z"
  name: mysecret
  namespace: default
  ownerReferences:
  - apiVersion: bitnami.com/v1alpha1
    controller: true
    kind: SealedSecret
    name: mysecret
    uid: c4e9b1f6-f7e6-4b74-83b5-610488110c9e
  resourceVersion: "6554569"
  selfLink: /api/v1/namespaces/default/secrets/mysecret
  uid: 46114282-adb9-4f32-887b-99a54d27ae76
type: Opaque
```