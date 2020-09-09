---
layout: default
---

## Gcloud Secret Manager for accessing environment variables in application


So finally Gcloud launched its own secret manager as a service. I got to know from Seth Vargo's [tweet](https://twitter.com/sethvargo/status/1220035296018018310), who btw, if you don't follow, has given some great talks around DevOps and generally on technology.
Before this, gcloud did not have anything around this to compete with [AWS Secret manager](https://aws.amazon.com/secrets-manager/) and [Hashicorp Vault](https://www.vaultproject.io/). Ofcourse you could use both of these and integrate with your gcloud resources. But after this, the integration is seemless.

Secret Manager allows you to store multiple version of a secret and only the permitted IAM role account can access or edit them, according to the permissions given to the role.

#### Step 1: Create a Secret

```bash
gcloud beta secrets create secret-key --replication-policy="automatic"
```

#### Step 2: Create a Version

A version is the secret value. Versioning is a good concept here in gcloud which can make rotation much easier.

* Open secret-key.txt in your favorite editor
* Type "xxx", the value of secret-key
* Save the file

```bash
gcloud beta secrets versions add "secret-key" --data-file secret-key.txt
```

#### Step 3: Access secret

```bash
gcloud beta secrets versions access 1 --secret="secret-key"
```

There is a bunch of other things also that you can easily do, like listing, describing and managing secrets and editing permissions of who can access them. There is official [documentation](https://cloud.google.com/secret-manager/docs/how-to) very well written.

Overall its a good choice if you are looking for a seamless effective secret manager to manage your applications environment variables for example. The price is $0.06 so you should keep in mind when you architect your infrastructure for example try to inject secrets in containers at deploy time rather than accessing them everytime at runtime.


[back](../)