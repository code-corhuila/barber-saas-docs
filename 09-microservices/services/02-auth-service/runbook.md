# Runbook — Auth Service

> **Framework example, not BarberSaaS's identity-auth.** It assumes Kubernetes and Redis;
> BarberSaaS runs on Docker Compose with no Redis (`05-architecture/deployment.md`). Use it as a
> template for the real `barber-saas-identity-auth-api` runbook.

> Operating procedures for whoever is on call.
> This service is critical — a P0 in auth-service prevents ALL users from accessing the system.

**Service:** auth-service
**Local port:** 8081
**Version:** 1.0
**Last updated:** [YYYY-MM-DD]

---

## 1. Quick information

| Field | Value |
|-------|-------|
| Local port | 8081 |
| Production URL | `https://api.[domain]/api/v1/auth` |
| Staging URL | `https://staging.api.[domain]/api/v1/auth` |
| Grafana dashboard | [URL of the auth-service panel] |
| Alert channel | [#alerts] |
| Escalation | [Tech Lead — contact — URGENT if auth is down] |
| Target RTO | 5 min (authentication blocked = system unusable) |
| Target RPO | 0 (PostgreSQL with real-time replica) |

---

## 2. Check health

```bash
# Health check
curl https://api.[domain]/api/v1/auth/health

# Expected response:
# {"status": "ok", "db": "connected", "redis": "connected"}

# If there are problems, check the dependencies:

# Check PostgreSQL
kubectl exec -n [ns] [postgres-pod] -- pg_isready -U [user]

# Check Redis
kubectl exec -n [ns] [redis-pod] -- redis-cli ping
# Expected response: PONG
```

---

## 3. Frequent alerts

### High rate of 401 errors on login

**Symptom:** Many failed login attempts in a short time (possible attack).

```bash
# See the IPs with the most attempts
kubectl logs -n [ns] -l app=auth-service --tail=500 | \
  grep '"eventType":"user.login_failed"' | jq '.payload.ipAddress' | \
  sort | uniq -c | sort -rn | head -10
```

**Action:** If an IP exceeds [N] attempts in 5 minutes, add it to the gateway's denylist.

### Refresh tokens do not work (users with an active session cannot renew)

**Probable cause:** Redis is down, or the JWT key was rotated without updating every service.

```bash
# Check Redis
kubectl exec -n [ns] [redis-pod] -- redis-cli ping

# Check that the public key in the gateway matches the auth-service one
kubectl exec -n [ns] [auth-pod] -- curl localhost:8081/api/v1/auth/jwks
```

### Accounts being locked out en masse

**Symptom:** Many users report "account locked" at the same time.

**Probable cause:** (A) a coordinated attack, (B) a counter bug that increments incorrectly.

```bash
# See how many accounts are currently locked
kubectl exec -n [ns] [redis-pod] -- redis-cli keys "locked:*" | wc -l

# Unlock a specific account (emergencies only, Tech Lead approval)
kubectl exec -n [ns] [redis-pod] -- redis-cli del "locked:user@example.com"
```

---

## 4. JWT key rotation

> Only run with the Tech Lead's approval. It impacts every service.

```bash
# 1. Generate a new key pair
openssl genrsa -out new-private.pem 2048
openssl rsa -in new-private.pem -pubout -out new-public.pem

# 2. Update the secret in the environment's vault
# (follow your infrastructure's vault process)

# 3. The JWKS endpoint supports several keys — add the new one without removing the old one
# This keeps existing tokens (signed with the old key) valid
# until they expire (max 1 hour)

# 4. After 1 hour: remove the old key from the JWKS

# 5. Check that the gateway can use the new public key
curl https://api.[domain]/api/v1/auth/jwks
```

---

## 5. Rollback

```bash
kubectl rollout history deployment/auth-service -n [ns]
kubectl rollout undo deployment/auth-service -n [ns]
kubectl rollout status deployment/auth-service -n [ns]
```

**If the deploy being rolled back included database migrations:**
Contact the Tech Lead before rolling back — a rollback migration may be required.

---

## 6. Maintenance operations

### Clean up expired refresh tokens

```bash
# If the automatic cleanup job fails:
kubectl exec -n [ns] [postgres-pod] -- psql -U [user] -d [db] \
  -c "DELETE FROM refresh_tokens WHERE expires_at < NOW() AND revoked_at IS NOT NULL;"
```

### Revoke all of a user's sessions (emergency)

```bash
kubectl exec -n [ns] [postgres-pod] -- psql -U [user] -d [db] \
  -c "UPDATE refresh_tokens SET revoked_at = NOW() WHERE user_id = '[user-uuid]';"
```

---

## 7. Post-incident

- [ ] Login and refresh working normally
- [ ] Zero 5xx in the last 5 minutes
- [ ] Incorrectly locked accounts unlocked (if applicable)
- [ ] Incident recorded in `13-operations/incident-management.md`
- [ ] If credentials were breached: escalate to the security protocol
