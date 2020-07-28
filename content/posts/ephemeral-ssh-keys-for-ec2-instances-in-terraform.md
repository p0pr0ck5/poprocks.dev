---
title: "Ephemeral SSH Keys for EC2 Instances in Terraform"
date: 2019-05-17T12:17:44-08:00
---

Terraform’s aws_key_pair resource requires an existing user-supplied key pair- it won’t create one for you. At first glance it would seem that leveraging this would require us to pre-generate a key pair outside of Terraform’s lifecycle, but we can do this natively with a bit of creative resource management.

<!--more-->

```hcl
resource "tls_private_key" "key" {
  algorithm = "RSA"
}

resource "local_file" "key_priv" {
  content  = "${tls_private_key.key.private_key_pem}"
  filename = "${path.module}/id_rsa"
}

resource "null_resource" "key_chown" {
  provisioner "local-exec" {
    command = "chmod 400 {path.module}/id_rsa"
  }

  depends_on = ["local_file.key_priv"]
}

resource "null_resource" "key_gen" {
  provisioner "local-exec" {
    command = "rm -f {path.module}/id_rsa.pub && ssh-keygen -y -f {path.module}/id_rsa > {path.module}/id_rsa.pub"
  }

  depends_on = ["local_file.key_priv"]
}

data "local_file" "key_pub" {
  filename = "${path.module}/id_rsa.pub"

  depends_on = ["null_resource.key_gen"]
}

resource "aws_key_pair" "key_tf" {
  public_key = "${data.local_file.key_pub.content}"
}

resource "tls_private_key" "key" {
  algorithm = "RSA"
}

resource "local_file" "key_priv" {
  content  = "${tls_private_key.key.private_key_pem}"
  filename = "${path.module}/id_rsa"
}

resource "null_resource" "key_chown" {
  provisioner "local-exec" {
    command = "chmod 400 {path.module}/id_rsa"
  }

  depends_on = ["local_file.key_priv"]
}

resource "null_resource" "key_gen" {
  provisioner "local-exec" {
    command = "rm -f {path.module}/id_rsa.pub && ssh-keygen -y -f {path.module}/id_rsa > {path.module}/id_rsa.pub"
  }

  depends_on = ["local_file.key_priv"]
}

data "local_file" "key_pub" {
  filename = "${path.module}/id_rsa.pub"

  depends_on = ["null_resource.key_gen"]
}

resource "aws_key_pair" "key_tf" {
  public_key = "${data.local_file.key_pub.content}"
}
```

Rather than generate an SSH key pair out of cycle, the tls_private_key resource can create an RSA key for us. We can then write it to disk and export the public key it to a format OpenSSH will like via the ssh-keygen utility. Once that’s done, we can read it back in as a data resource (taking care to depend on it) and add the AWS resource. Following all that, we can apply the key pair to instances, ASGs, etc:


```hcl
resource "aws_instance" "instance" {
  key_name = "${aws_key_pair.key_tf.key_name}"
}

resource "aws_instance" "instance" {
  key_name = "${aws_key_pair.key_tf.key_name}"
}
```
