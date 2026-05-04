# DBS302 – NoSQL Database Management
## Practical 6: Securing Redis and MongoDB
### Authentication, Encryption, RBAC, and Security Audit

**Student:** Kuenzang Rabten  
**Date:** May 4, 2026

---

## 1. Aim and Objectives

The aim of this practical is to configure and verify authentication, encryption, and role-based access control (RBAC) for Redis and MongoDB, and to perform a basic security audit of both databases.

By the end of this practical I am able to:

- Enable password-based authentication and Access Control Lists (ACL) in Redis
- Enable TLS encryption for Redis connections
- Enable authentication and RBAC in MongoDB using built-in and custom roles
- Enable TLS for MongoDB so all traffic is encrypted in transit
- Execute a security audit checklist for both databases and record findings

---

## 2. Short Theory

Three key security ideas apply to both Redis and MongoDB:

- **Authentication** – Proving who a client is. Both Redis and MongoDB require a username and password before any command can run. Without valid credentials the connection is rejected.
- **Encryption (TLS)** – Protecting data while it moves over the network. Using self-signed certificates we make both Redis and MongoDB listen only on TLS, so plain-text connections are refused.
- **Role-Based Access Control (RBAC)** – Giving each user the minimum permissions they need. Redis uses ACL rules (key patterns + allowed commands). MongoDB uses roles that list specific privileges on specific databases and collections.

---

## 3. Part A - Securing Redis

### 3.1 Step 1: Confirm Redis is Running

I started Redis with the lab configuration file and confirmed it was listening with a PING test.

```bash
redis-server /etc/redis/redis.conf
redis-cli ping      # returns PONG
```

### 3.2 Step 2: Enable ACL Users (redis.conf)

I added three ACL entries to `redis.conf` to disable the anonymous default user and create three named users with different permission levels:

```bash 
user default off
user admin on >adminStrongPwd ~* +@all
user app_user on >appStrongPwd ~session:* +get +set +del +expire +ttl +@connection
user monitoring on >monitorPwd ~* +@read +info +dbsize +lastsave +@connection
```

After restarting Redis I tested each user. The screenshot below shows `app_user` succeeding on `session:*` keys but getting a permission error when trying to write to a different key pattern (RBAC enforced).After restarting Redis I tested each user. The screenshot below shows `app_user` succeeding on `session:*` keys but getting a permission error when trying to write to a different key pattern (RBAC enforced).

![alt text](<assets/Screenshot from 2026-05-04 16-49-28.png>)

I then checked the full ACL list as admin and confirmed all four user entries were correctly stored.

![alt text](<assets/Screenshot from 2026-05-04 16-55-38.png>)

### 3.3 Step 3: Enable TLS for Redis

I generated a self-signed CA and server certificate under `/etc/redis/tls/` using `openssl`.

![alt text](<assets/Screenshot from 2026-05-04 16-59-07.png>)

Then I updated `redis.conf` to set `port 0` (disable plain TCP) and `tls-port 6379` with the certificate files. After restart I tested both a TLS connection (success) and a plain-text connection (refused).

![alt text](<assets/Screenshot from 2026-05-04 17-01-31.png>)

---

## 4. Part B – Securing MongoDB

### 4.1 Step 1: Create First Admin User

I started `mongod` without auth to create the initial `rootAdmin` user in the `admin` database.

![alt text](<assets/Screenshot from 2026-05-04 17-02-41.png>)

Then I added `security.authorization: enabled` to `mongod.conf` and restarted MongoDB so all subsequent connections require authentication.

### 4.2 Step 2: RBAC – Custom Role and Application User

Authenticated as `rootAdmin` I created a custom role (`myAppRole`) that only allows CRUD on `myapp.customers`. I then created `appUser` with that role. The screenshot shows `appUser` succeeding on the `customers` collection but getting an authorization error when trying to query the `admin` database.

![alt text](<assets/Screenshot from 2026-05-04 17-04-44.png>)

### 4.3 Step 3: Enable TLS for MongoDB

I generated a CA and server certificate under `/etc/mongo/tls/` and combined the key and cert into `mongo.pem`. I updated `mongod.conf` with `net.tls.mode: requireTLS`. The screenshot below shows both the certificate generation steps and a successful TLS-authenticated connection inserting a document.

![alt text](<assets/Screenshot from 2026-05-04 17-06-04.png>)

---

## 5. Part C - Security Audit Summary

After completing configuration I ran a structured audit to test both successful and denied operations on each database. The table below summarises findings:

| Check | Command / Test | Result |
|---|---|---|
| Redis – anon access | `redis-cli ping` (no credentials) | PASS – NOAUTH error |
| Redis – ACL key scope | `app_user set otherkey` | PASS – NOPERM error |
| Redis – dangerous cmd | `app_user FLUSHALL` | PASS – NOPERM error |
| Redis – monitoring write | `monitoring set blockme` | PASS – NOPERM error |
| Redis – TLS enforced | plain-text connect on port 6379 | PASS – connection reset |
| MongoDB – anon access | `mongosh` without credentials | PASS – auth error |
| MongoDB – RBAC cross-DB | `appUser` access admin DB | PASS – not authorized |
| MongoDB – TLS enforced | `mongosh` without `--tls` flag | PASS – TLS required |

![alt text](<assets/Screenshot from 2026-05-04 17-07-50.png>)

---

## 6. Conclusion

This practical showed me how to properly lock down two NoSQL databases that are often left open by default.

- For **Redis** I used ACL to create scoped users and restricted commands and key patterns, then added TLS to stop anyone reading traffic on the wire.
- For **MongoDB** I enabled authorization so every connection must log in, created a focused custom role for the application user so it can only touch the right collection, and then layered TLS on top.

The security audit confirmed that every negative case was correctly rejected. The main things still worth improving in a real setup would be:

- Using stronger passwords
- Binding to specific IPs instead of `0.0.0.0`
- Using proper CA-signed certificates instead of self-signed ones