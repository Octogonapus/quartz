---
date: 2024-07-30
description: An overview of how to connect to RDS and RDS Proxy.
---

An overview of how to connect to RDS and RDS Proxy because I often see people getting confused.

## RDS Instance

Connecting directly to an RDS instance requires using the certificate authority (CA) matching that of database you are connecting to (e.g. `rds-ca-2019`, `rds-ca-rsa2048-g1`).[^rds-with-ssl] If you are using `rds-ca-2019` you also need the intermediate certificates, not just the root certificates.[^rds-with-ssl] AWS provides global and regional bundles for this use case; download them [here](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html#UsingWithRDS.SSL.CertificatesDownload).

## RDS Proxy

Connecting to an RDS proxy requires using the certificate authority (CA) matching that of RDS proxy, not the CA of the RDS database you are connecting to via the proxy.[^rds-proxy] AWS provides the certificate authorities used for RDS proxy via [AWS Trust Services](https://www.amazontrust.com/repository/); download them [here](https://www.amazontrust.com/repository/). Download all root CAs and place them into a single `.pem` file.[^rds-proxy] 

## RDS Instance via Jump box

Connecting to an RDS instance via a jump box (via an SSH tunnel) works the same as connecting to an RDS instance directly for SSL modes `PREFERRED`, `REQUIRED`, and `VERIFY_CA` (i.e. use the global/regional bundle). SSL mode `VERIFY_IDENTITY` (for MySQL, also called `VERIFY_FULL` for PostgreSQL) will not work unless you add a hostname mapping to `/etc/hosts` (assuming Linux) and then have MySQL connect to that hostname. Here is an example:

```sh
echo '127.0.0.1	my-db.foo.us-east-1.rds.amazonaws.com' | sudo tee -a /etc/hosts
ssh -N -L 3306:my-db.foo.us-east-1.rds.amazonaws.com:3306 ubuntu@123.123.123.123 -i ~/.ssh/my-instance-key.pem &
mysql -u admin -padminadmin -h my-db.foo.us-east-1.rds.amazonaws.com --ssl-mode=VERIFY_IDENTITY --ssl-ca=global-bundle.pem
```

## RDS Proxy via Jump box

Connecting to an RDS proxy via a jump box (via an SSH tunnel) works the same as connecting to an RDS proxy directly for SSL modes `PREFERRED`, `REQUIRED`, and `VERIFY_CA` (i.e. use the AWS Trust Services root CAs). SSL mode `VERIFY_IDENTITY` requires the same hostname trick as above.

## RDS Instance And/Or RDS Proxy

There is nothing preventing you from putting both the [global bundle](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html#UsingWithRDS.SSL.CertificatesDownload) and [trust services root CAs](https://www.amazontrust.com/repository/) in the same `.pem` file if you need to connect to both and RDS instance and an RDS proxy.

[^rds-with-ssl]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.SSL.html
[^rds-proxy]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.howitworks.html#rds-proxy-security.tls
[^rds-proxy-init]: During initial creation, RDS proxy may transition a target group between the states `UNAVAILABLE` due to insufficient capacity -> `UNAVAILABLE` due to an internal error -> `AVAILABLE`. Wait up to 20 minutes before concluding the proxy has failed to initialize.
